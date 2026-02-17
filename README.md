# DeepSeek-OCR GGUF (for llama.cpp PR #17400)

This repository provides GGUF model files required for running the DeepSeek-OCR / MTMD support introduced in the following llama.cpp pull request:

> [llama.cpp PR #17400](https://github.com/ggml-org/llama.cpp/pull/17400)

**These models are only compatible with the PR branch and will not run on upstream llama.cpp main.**

## Model Files

| File | Size | Description |
|------|------|-------------|
| `deepseek-ocr-f16.gguf` | 5.5 GB | Language model (BF16 full precision) |
| `deepseek-ocr-Q4_K_M.gguf` | 1.8 GB | Language model (Q4_K_M quantized) |
| `mmproj-deepseek-ocr-f16.gguf` | 774 MB | Vision projector (f16, always required) |

The language model uses a Mixture-of-Experts (MoE) architecture: 64 experts, 6 active per token, ~2.93B total parameters. The vision encoder uses a 1024x1024 image size with 16x16 patches.

**Note:** The f16 model supports all prompt modes reliably. The Q4_K_M quantization is faster and smaller but several prompts are unstable with it (see [Known Issues](#known-issues)).

## Prerequisites

- **cmake** (tested with 4.2.3)
- **C++ compiler** with C++17 support
- **Git** and optionally **GitHub CLI** (`gh`)
- **Python 3.11+** with `huggingface_hub[cli]` (for model download)

### Python Environment (recommended)

```bash
conda create -n deepseek-ocr python=3.11 -y
conda activate deepseek-ocr
pip install "huggingface_hub[cli]"
```

## Download

GGUF files are available from [sabafallah/DeepSeek-OCR-GGUF](https://huggingface.co/sabafallah/DeepSeek-OCR-GGUF) on Hugging Face.

Available quantizations:

| Quantization | Size | Notes |
|---|---|---|
| Q4_K_M | 1.95 GB | Good balance of quality and size (recommended) |
| Q8_0 | 3.13 GB | Higher quality, more RAM |
| F16 | 5.88 GB | Full precision |

The vision projector (`mmproj`) is always needed in f16 regardless of model quantization.

Download using `huggingface-cli`:

```bash
# Language model (pick one quantization)
huggingface-cli download sabafallah/DeepSeek-OCR-GGUF \
  --include "deepseek-ocr-Q4_K_M.gguf" \
  --local-dir gguf_models/deepseek-ai

# Vision projector (always required)
huggingface-cli download sabafallah/DeepSeek-OCR-GGUF \
  --include "mmproj-deepseek-ocr-f16.gguf" \
  --local-dir gguf_models/deepseek-ai
```

Alternative: [NexaAI/DeepSeek-OCR-GGUF](https://huggingface.co/NexaAI/DeepSeek-OCR-GGUF) offers additional quantizations (Q4_0, Q5_0, Q6_K) but without mmproj files.

## Build llama.cpp PR Branch

Clone llama.cpp and check out the PR branch:

```bash
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp

# Checkout PR branch (recommended: GitHub CLI)
gh pr checkout 17400
# or manually:
# git fetch origin pull/17400/head:pr17400
# git checkout pr17400
```

Build:

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
```

The build auto-detects available backends:
- **macOS Apple Silicon**: Metal GPU backend (automatic)
- **Linux with NVIDIA GPU**: add `-DGGML_CUDA=ON`
- **Linux CPU-only**: works out of the box, optionally add `-DGGML_BLAS=ON` for OpenBLAS acceleration

## Usage

### The 6 Official Prompts

DeepSeek-OCR supports six documented prompt modes. All examples use `--chat-template deepseek-ocr --temp 0`.

#### 1. `<|grounding|>Convert the document to markdown.` — Document OCR with layout (recommended)

```bash
llama-mtmd-cli \
  -m gguf_models/deepseek-ai/deepseek-ocr-Q4_K_M.gguf \
  --mmproj gguf_models/deepseek-ai/mmproj-deepseek-ocr-f16.gguf \
  --image document.png \
  -p "<|grounding|>Convert the document to markdown." \
  --chat-template deepseek-ocr --temp 0
```

Returns text with positional annotations for each recognized block. Can produce Markdown headings (`#`, `##`) for documents with heading hierarchy:

```
<|ref|>title<|/ref|><|det|>[[73, 167, 278, 222]]<|/det|>
Rechnung

<|ref|>text<|/ref|><|det|>[[73, 229, 287, 250]]<|/det|>
Rechnungsnummer: 5462339178
```

Best for: **document OCR** — the most complete and accurate mode. Only mode that reliably captures multi-column layouts. Bounding box tags can be stripped in post-processing for clean Markdown.

#### 2. `<|grounding|>OCR this image.` — Compact OCR with positions

```bash
llama-mtmd-cli \
  -m gguf_models/deepseek-ai/deepseek-ocr-Q4_K_M.gguf \
  --mmproj gguf_models/deepseek-ai/mmproj-deepseek-ocr-f16.gguf \
  --image document.png \
  -p "<|grounding|>OCR this image." \
  --chat-template deepseek-ocr --temp 0
```

More compact format with text directly inside `<|ref|>` tags:

```
<|ref|>Google Cloud EMEA Limited<|/ref|><|det|>[[740, 97, 935, 117]]<|/det|>
<|ref|>BERND STREBEL<|/ref|><|det|>[[66, 347, 188, 366]]<|/det|>
```

Best for: spatial text extraction. **Caveat:** This prompt mode loses umlauts and special characters regardless of quantization ("Jagerlauf" instead of "Jägerlauf", "gemal3"/"gemai3" instead of "gemäß"). This is a prompt-level limitation, not a quantization issue — both Q4_K_M and f16 produce the same errors.

#### 3. `Convert the document to markdown.` — Plain text (no grounding)

```bash
llama-mtmd-cli \
  -m gguf_models/deepseek-ai/deepseek-ocr-Q4_K_M.gguf \
  --mmproj gguf_models/deepseek-ai/mmproj-deepseek-ocr-f16.gguf \
  --image document.png \
  -p "Convert the document to markdown." \
  --chat-template deepseek-ocr --temp 0
```

Returns clean text without positional data. ~2.5x faster than grounding mode but may miss right-aligned or multi-column content.

Best for: quick text extraction where layout information is not needed.

#### 4. `Free OCR.` — Fast text extraction (f16 only)

```bash
llama-mtmd-cli \
  -m gguf_models/deepseek-ai/deepseek-ocr-f16.gguf \
  --mmproj gguf_models/deepseek-ai/mmproj-deepseek-ocr-f16.gguf \
  --image document.png \
  -p "Free OCR." \
  --chat-template deepseek-ocr --temp 0
```

Produces clean text output, identical to plain text mode in our testing. **Requires the f16 model** — Q4_K_M produces an infinite loop with this prompt.

#### 5. `Parse the figure.` — Charts, diagrams, tables

```bash
llama-mtmd-cli \
  -m gguf_models/deepseek-ai/deepseek-ocr-Q4_K_M.gguf \
  --mmproj gguf_models/deepseek-ai/mmproj-deepseek-ocr-f16.gguf \
  --image figure.png \
  -p "Parse the figure." \
  --chat-template deepseek-ocr --temp 0
```

Designed for charts, diagrams, and tables. Can produce HTML tables (`<td>`), SMILES notation (chemistry), or structured descriptions depending on content type. When used on a regular document, produces a natural language description instead of OCR text. **Caution:** May hallucinate content not present in the image (observed with both Q4_K_M and f16).

Best for: **charts, diagrams, and tables** — not for standard document OCR.

#### 6. `Describe this image in detail.` — Image understanding

```bash
llama-mtmd-cli \
  -m gguf_models/deepseek-ai/deepseek-ocr-Q4_K_M.gguf \
  --mmproj gguf_models/deepseek-ai/mmproj-deepseek-ocr-f16.gguf \
  --image document.png \
  -p "Describe this image in detail." \
  --chat-template deepseek-ocr --temp 0
```

Generates a natural language description of the image in English. Not an OCR mode — describes visual layout, colors, and content structure rather than extracting text verbatim.

Best for: **image understanding and captioning**, not text extraction.

### Additional Prompt: Text Localization

```bash
llama-mtmd-cli \
  -m gguf_models/deepseek-ai/deepseek-ocr-f16.gguf \
  --mmproj gguf_models/deepseek-ai/mmproj-deepseek-ocr-f16.gguf \
  --image document.png \
  -p "Locate <|ref|>Google Cloud<|/ref|> in the image." \
  --chat-template deepseek-ocr --temp 0
```

Returns bounding box coordinates for all text regions in the document. **Requires f16 model** — Q4_K_M produces an infinite loop. Note: In testing, the output contained only bounding boxes without extracted text content.

While not a direct OCR mode, the Locate prompt is useful in OCR workflows for:
- **Annotation**: drawing bounding boxes or highlights around specific text regions
- **Validation**: verifying whether a specific text string appears in a document
- **Field extraction**: locating and cropping specific areas (e.g. invoice numbers, signatures) for downstream processing
- **Search & highlight**: visual search across document batches

### Prompt Compatibility Matrix

| Prompt | Q4_K_M | f16 | Output Type |
|--------|--------|-----|-------------|
| `<\|grounding\|>Convert the document to markdown.` | OK | OK | Markdown + bounding boxes |
| `<\|grounding\|>OCR this image.` | OK (umlaut issues) | OK (same umlaut issues) | Text + bounding boxes |
| `Convert the document to markdown.` | OK | OK | Plain text |
| `Free OCR.` | Infinite loop | OK | Plain text |
| `Parse the figure.` | OK (may hallucinate) | OK (may hallucinate) | Description / HTML tables |
| `Describe this image in detail.` | OK | OK | English description |
| `Locate <\|ref\|>...<\|/ref\|>` | Infinite loop | OK | Bounding boxes (no text) |

### Input Format

- **Supported**: PNG, JPG/JPEG image files
- **Not supported**: PDF files (convert to images first, e.g. with `pdftoppm -png -r 300 document.pdf output`)

## Test Results

Tested on macOS with Apple M3 Max, using a Google Cloud invoice (German) as input.

### Performance Comparison

| Metric | Grounding (Q4_K_M) | Plain Text (Q4_K_M) | Free OCR (f16) |
|--------|-------------------|---------------------|----------------|
| Output | Text + bounding boxes | Clean text only | Clean text only |
| Tokens generated | 890 | 259 | 259 |
| Total time | 5.6 sec | 2.2 sec | 3.5 sec |
| Prompt eval | 199 tokens/sec | 386 tokens/sec | 284 tokens/sec |
| Generation | 340 tokens/sec | 326 tokens/sec | 215 tokens/sec |
| GPU RAM | ~2.5 GB | ~2.5 GB | ~6.0 GB |
| Completeness | Full document | Partial | Partial |

**Key findings:**
- **Grounding mode** (`<|grounding|>Convert the document to markdown.`) is the most complete and accurate mode — only mode that reliably captures multi-column layouts (e.g. right-aligned sender address)
- **f16 detects finer document structure** than Q4_K_M (e.g. `sub_title` tag for "Google Cloud" vs. plain `text`)
- **Plain text mode**, **Free OCR**, and **Convert to markdown (no grounding)** produce identical output — all may omit right-aligned or secondary content blocks
- **`<|grounding|>OCR this image.`** has umlaut/special character issues regardless of quantization — this is a prompt-level limitation
- **`Parse the figure.`** may hallucinate content not present in the image (both Q4_K_M and f16)
- **Q4_K_M** is faster and more memory-efficient, but "Free OCR." and "Locate" prompts produce infinite loops
- **f16** supports all prompt modes reliably but uses ~3.5 GB more GPU RAM

### OCR Quality

- All German umlauts (ä, ö, ü) and special characters (€) correctly recognized
- Addresses, invoice numbers, tax IDs, and amounts accurately extracted
- Multi-column layout correctly parsed in grounding mode
- Reverse-charge tax notice fully captured

### Platform Compatibility

| Platform | Backend | Expected Performance |
|----------|---------|---------------------|
| macOS Apple Silicon | Metal (GPU) | ~284-386 tokens/sec prompt eval |
| Linux with NVIDIA GPU | CUDA | Comparable to Metal |
| Linux CPU-only | AVX2/AVX-512 | ~20-80 tokens/sec (CPU dependent) |

RAM requirement: ~2.6 GB (Q4_K_M) or ~6.3 GB (f16) including vision projector.

## Known Issues

1. **Q4_K_M + "Free OCR." / "Locate" prompts** — produce infinite loops of dots (`. . . . .`) until the context window is exhausted. Use the f16 model for these prompts, or use grounding/plain text mode with Q4_K_M
2. **`<|grounding|>OCR this image.` loses umlauts** — This is a prompt-level limitation, not a quantization issue. Both Q4_K_M and f16 produce the same errors ("Jagerlauf" instead of "Jägerlauf", "gemal3" instead of "gemäß"). Use `<|grounding|>Convert the document to markdown.` instead for correct character recognition
3. **`Parse the figure.` may hallucinate** — When used on regular documents (not charts/tables), both Q4_K_M and f16 may generate descriptions containing text not present in the original image
4. **Plain text / Free OCR / Convert (no grounding)** may omit right-aligned or multi-column content — use grounding mode for complete extraction
5. **No N-gram penalty in llama.cpp** — The official Python implementation uses an N-gram penalty processor (`ngram_size=30`, `window_size=90`) to prevent repetition loops. llama.cpp does not implement this, which likely contributes to the infinite loop issues with Q4_K_M
6. **PDF input** is not directly supported — pre-convert pages to PNG/JPG images (e.g. `pdftoppm -png -r 300 document.pdf output`)

## Project Structure

```
DeepSeekOCR/
├── README.md
├── gguf_models/
│   └── deepseek-ai/
│       ├── deepseek-ocr-f16.gguf          # Language model (f16, 5.5 GB)
│       ├── deepseek-ocr-Q4_K_M.gguf      # Language model (Q4_K_M, 1.8 GB)
│       └── mmproj-deepseek-ocr-f16.gguf   # Vision projector (774 MB)
└── llama.cpp/                              # PR #17400 branch build
    └── build/bin/
        └── llama-mtmd-cli                  # OCR executable
```

## License

DeepSeek-OCR model: [MIT License](https://github.com/deepseek-ai)
llama.cpp: [MIT License](https://github.com/ggml-org/llama.cpp/blob/master/LICENSE)
