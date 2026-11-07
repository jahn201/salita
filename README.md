# Salita

**Voice typing for Windows that understands Tagalog, English, and Taglish.**

Hold **Ctrl + Win**, speak, let go. Salita transcribes your voice on your own GPU, cleans it up with Claude (punctuation, filler words, misheard words), and pastes the text into whatever app you're using.

> Status: **planning / early build.** See [docs/BUILD_PLAN.md](docs/BUILD_PLAN.md) for the roadmap.

## How it works

```
Hold Ctrl+Win → record mic → whisper.cpp (local GPU) → Claude Haiku cleanup → paste into active window
```

- **Speech-to-text runs locally** with [whisper.cpp](https://github.com/ggml-org/whisper.cpp) (`large-v3-turbo`, ~0.9 GB). Works on NVIDIA (CUDA), AMD (Vulkan), or CPU.
- **Cleanup uses Claude** via your own Anthropic API key. Only text is sent, never audio.
- **Taglish stays Taglish.** Salita doesn't translate what you say unless you ask it to.

## Requirements

- Windows 10/11
- An Anthropic API key
- Recommended: a dedicated GPU (NVIDIA or AMD) with 2 GB+ free VRAM

## Install

Coming in the first release. See the build plan for progress.

## License

[MIT](LICENSE)
