# DeepSeekOCR - Project Instructions

## Project Overview

This project evaluates DeepSeek-OCR running locally via llama.cpp (PR #17400 branch). It is not a library or application — it is a documentation and experimentation project for GGUF-based OCR inference.

## Repository Structure

- `README.md` — comprehensive documentation of setup, prompts, test results, and known issues
- `.gitignore` — excludes model files, llama.cpp build, test images, and .claude/
- `gguf_models/deepseek-ai/` — GGUF model files (not in git, ~8 GB total)
- `llama.cpp/` — cloned from PR #17400 branch `sf/deepseek-ocr` (not in git)

## Key Paths

- **Binary**: `llama.cpp/build/bin/llama-mtmd-cli`
- **Q4_K_M model**: `gguf_models/deepseek-ai/deepseek-ocr-Q4_K_M.gguf` (1.8 GB)
- **f16 model**: `gguf_models/deepseek-ai/deepseek-ocr-f16.gguf` (5.5 GB)
- **Vision projector**: `gguf_models/deepseek-ai/mmproj-deepseek-ocr-f16.gguf` (774 MB)

## Running OCR

All commands require `--chat-template deepseek-ocr --temp 0`.

Best prompt for document OCR:
```bash
llama.cpp/build/bin/llama-mtmd-cli \
  -m gguf_models/deepseek-ai/deepseek-ocr-Q4_K_M.gguf \
  --mmproj gguf_models/deepseek-ai/mmproj-deepseek-ocr-f16.gguf \
  --image <image.png> \
  -p "<|grounding|>Convert the document to markdown." \
  --chat-template deepseek-ocr --temp 0
```

## Important Notes

- **Do not commit test images** — they may contain personal data (invoices, IDs)
- **Do not commit GGUF files** — too large for git, download from HuggingFace
- **Q4_K_M is unstable** with "Free OCR." and "Locate" prompts — use f16 for these
- **`<|grounding|>OCR this image.`** loses umlauts/special chars — this is a prompt-level bug, not quantization
- **`Parse the figure.`** can hallucinate content — verify output against source
- **Recommended prompt**: `<|grounding|>Convert the document to markdown.` with Q4_K_M for best balance of speed, quality, and completeness

## Conda Environment

```bash
conda activate deepseek-ocr
```

Python 3.11 with `huggingface_hub` installed. Used for model downloads via:
```python
from huggingface_hub import hf_hub_download
```

## Rebuilding llama.cpp

If llama.cpp needs to be rebuilt (e.g. after PR update):
```bash
cd llama.cpp
git pull
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
```

## External References

- [llama.cpp PR #17400](https://github.com/ggml-org/llama.cpp/pull/17400)
- [DeepSeek-OCR official repo](https://github.com/deepseek-ai/DeepSeek-OCR)
- [GGUF models on HuggingFace](https://huggingface.co/sabafallah/DeepSeek-OCR-GGUF)
- [DeepSeek-OCR prompt guide](https://deepwiki.com/deepseek-ai/DeepSeek-OCR/3.4-working-with-prompts)
