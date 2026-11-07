# Salita build plan

A personal, open-source Wispr Flow alternative for Windows that handles Tagalog, English, and Taglish.

This file is the brief for the build sessions. Work through the phases in order. Each phase has a "done when" check: don't move on until it passes.

## Decisions (locked)

| Area | Decision | Notes |
|---|---|---|
| OS | Windows 10/11 only | System tray app |
| Language | Python 3.12+ (dev machine has 3.14) | Check that every dependency has wheels for the Python version used |
| Hotkey | Hold **Ctrl + Win** to talk, release to stop | Same as Wispr Flow on Windows |
| Speech-to-text | whisper.cpp `whisper-server.exe`, run as a local subprocess | Model stays loaded so each request is fast |
| Model | `ggml-large-v3-turbo-q5_0.bin` (~550 MB) or `ggml-large-v3-turbo.bin` (~1.6 GB) | Start with q5_0; compare accuracy on the test set |
| GPU backends | CUDA (NVIDIA), Vulkan (AMD, e.g. RX 9070 XT), CPU fallback | Auto-detect GPU vendor |
| Cleanup LLM | Claude Haiku 5.5 (`claude-haiku-5-5`) via the `anthropic` SDK | User supplies their own API key. Only text is sent |
| Output | Clipboard + simulated Ctrl+V, then restore the old clipboard | |
| License | MIT, public repo | |

### Why whisper.cpp and not faster-whisper
faster-whisper (CTranslate2) only accelerates on NVIDIA. whisper.cpp supports Vulkan, which covers AMD cards on Windows without ROCm.

### Vulkan binaries
whisper.cpp releases ship Windows CPU and CUDA zips but **no Windows Vulkan build**. Plan: a GitHub Actions workflow in this repo (`.github/workflows/whisper-vulkan.yml`) that:
1. Checks out `ggml-org/whisper.cpp` at a pinned tag.
2. Installs the Vulkan SDK (e.g. `jakoch/install-vulkan-sdk-action`).
3. Builds with `cmake -B build -DGGML_VULKAN=ON -DWHISPER_BUILD_SERVER=ON` and `cmake --build build --config Release`.
4. Uploads `whisper-server.exe` and its DLLs as a release asset of this repo (`whisper-vulkan-win-x64.zip`).

Building locally also works: install Visual Studio Build Tools (C++), CMake, and the Vulkan SDK, then run the same cmake commands.

## Project layout

```
salita/
  salita/
    __main__.py        # entry point: python -m salita
    app.py             # wires everything together, state machine
    hotkey.py          # low-level keyboard hook (Ctrl+Win hold-to-talk)
    audio.py           # mic capture with sounddevice, 16 kHz mono
    stt.py             # starts/stops whisper-server, sends audio, returns text
    cleanup.py         # Claude prompt + call
    paste.py           # clipboard save / set / Ctrl+V / restore
    tray.py            # pystray icon, menu, recording indicator
    overlay.py         # small "listening..." pill near the bottom of the screen (phase 4)
    config.py          # settings in %APPDATA%\Salita\config.json, API key in Windows Credential Manager (keyring)
    history.py         # last N dictations (raw + cleaned) for debugging
  scripts/
    setup_whisper.py   # detect GPU, download the right whisper.cpp build + model into %LOCALAPPDATA%\Salita
  tests/
    samples/           # your recorded test sentences (not committed if private)
    eval_stt.py        # runs the test set, prints raw vs cleaned vs expected
  docs/BUILD_PLAN.md
  pyproject.toml
```

## State machine

```
IDLE --(Ctrl+Win held > 150 ms)--> RECORDING
RECORDING --(either key released)--> TRANSCRIBING
RECORDING --(held < 150 ms or audio < 0.3 s)--> IDLE (discard)
TRANSCRIBING --> CLEANING --> PASTING --> IDLE
any error --> IDLE + tray notification
```

Only one dictation at a time. Recording again while still processing is ignored (or queued, decide in phase 4).

## Phase 1: tray, hotkey, recording

**Build**
- `hotkey.py`: `SetWindowsHookExW(WH_KEYBOARD_LL)` via `ctypes`, on its own thread with a message loop. Track `VK_LCONTROL`/`VK_RCONTROL` and `VK_LWIN`/`VK_RWIN`.
- **Start menu suppression:** when Win is released after being used for dictation, Windows opens the Start menu. Before letting the Win key-up through, inject a dummy key press (`VK_NONAME` 0xFC, or an unassigned key like 0xE8) with `SendInput`. Windows then treats it as a chord and does not open Start.
- Don't swallow the keys themselves. Other shortcuts (Win+Ctrl+D, Win+Ctrl+arrows for virtual desktops) must still work. If another key is pressed while Ctrl+Win is held, cancel the recording.
- `audio.py`: `sounddevice.InputStream` at 16 kHz, mono, int16, written to an in-memory buffer. Start on RECORDING, stop on release.
- `tray.py`: `pystray` icon with a different color while recording. Menu: Settings (placeholder), Open history, Quit.

**Done when:** holding Ctrl+Win shows the recording icon, releasing saves a playable WAV, and the Start menu does **not** open.

## Phase 2: local whisper transcription

**Build**
- `scripts/setup_whisper.py`:
  - Detect the GPU with `Get-CimInstance Win32_VideoController` (or WMI).
  - NVIDIA → download `whisper-bin-win-cuda-12.4.0-x64.zip` from the latest whisper.cpp release that has assets (the tagged `v1.x` releases may have none; the `bXXXX` builds do).
  - AMD / Intel → download this repo's Vulkan zip.
  - Otherwise → `whisper-blas-bin-x64.zip` (CPU).
  - Download the model from Hugging Face `ggerganov/whisper.cpp`. Verify the SHA.
  - Everything goes into `%LOCALAPPDATA%\Salita\bin` and `\models`.
  - Check that the zip actually contains `whisper-server.exe`. If not, use `whisper-cli.exe` per request as a fallback (slower: reloads the model each time).
- `stt.py`:
  - Start `whisper-server.exe -m <model> --host 127.0.0.1 --port <free port> -l auto` on app start. Wait for it to be ready.
  - POST the WAV to `/inference` with `response_format=json`, `temperature=0`.
  - Language: `auto` picks one language per clip, which can hurt Taglish. Test `auto` vs forcing `tl` vs `en` and pick the best default. Make it a setting.
  - Pass an initial prompt (`prompt` field) containing a short Taglish example and the user's dictionary words. This strongly helps Whisper with code-switching and spelling.
  - Kill the server on quit.

**Done when:** you speak a Taglish sentence and the raw transcript appears in the console in under ~1 s on GPU.

## Phase 3: Claude cleanup

**Build**
- `cleanup.py` using the `anthropic` SDK, model `claude-haiku-5-5`, `max_tokens` ~ 2x the transcript length, low temperature.
- API key stored with `keyring` (Windows Credential Manager), never in a plain file.
- If the API call fails or times out (~4 s), paste the raw transcript instead of nothing.
- Skip cleanup for very short clips (1-2 words) to save latency.

**Draft system prompt** (tune with the test set):

```
You clean up voice dictation for a bilingual Filipino speaker who mixes Tagalog and English (Taglish).

Rules:
- Output ONLY the cleaned text. No quotes, no explanations.
- Keep the speaker's language exactly as spoken. Do NOT translate Tagalog to English or English to Tagalog. Taglish stays Taglish.
- Use standard Tagalog spelling (e.g. "nang" vs "ng", "din/rin", "daw/raw") and correct English spelling.
- Add punctuation and capitalization.
- Remove filler words and false starts: uhm, ah, eh, ano (when used as a filler), parang (when filler), like (when filler), you know.
- If the speaker corrects themself ("bukas, ay hindi, sa Friday pala"), keep only the final intent.
- Fix words the speech recognizer clearly misheard, using context. Prefer words from the user's dictionary.
- Keep the speaker's tone. Don't make it more formal unless the mode says so.
- Format spoken lists as lists only when the speaker clearly lists items.

Mode: {mode}
User dictionary: {dictionary}
```

Modes: `casual` (default), `formal` (emails), `raw` (punctuation only, minimal changes).

**Done when:** across the test set, cleaned output reads like something you typed, with no translating and no added content.

## Phase 4: paste, settings, polish

- `paste.py`: save the clipboard (text formats at least), set the cleaned text, `SendInput` Ctrl+V, wait ~150 ms, restore the old clipboard. Make sure Ctrl and Win are physically released before sending Ctrl+V.
- Overlay pill ("Listening..." / "Thinking...") near the bottom center, click-through, always on top (tkinter or a tiny win32 layered window).
- Settings window: API key, mode, language, dictionary words, microphone choice, start with Windows.
- History: last 50 entries (raw, cleaned, time, latency) in `%APPDATA%\Salita\history.jsonl`, with "copy again" from the tray.
- Start with Windows: shortcut in the Startup folder or `HKCU\...\Run`.
- Packaging: PyInstaller one-folder build, then an Inno Setup installer. Release via GitHub Actions.

**Done when:** works in Notepad, Chrome, Discord, VS Code, and Messenger; your clipboard comes back intact; it survives a reboot.

## Phase 5 (optional)

- Command mode: select text, hold Ctrl+Win+Alt, say "translate to English" / "gawing formal" / "summarize", and Claude rewrites the selection.
- Per-app modes (formal in Outlook, casual in Messenger).
- Online STT engine option (Groq / OpenAI) for people without a GPU, if they add their own key.
- Voice-activity trimming of silence before sending to Whisper.

## Test set and latency targets

Record 15-20 sentences the way you actually speak, saved as `tests/samples/NN.wav` with the expected text in `NN.txt`. Include:
- pure Tagalog, pure English, heavy Taglish in one sentence
- names, places, brand/slang words from your dictionary
- fillers and self-corrections
- a long rambling paragraph (30+ s)

`eval_stt.py` prints raw transcript, cleaned text, and expected text side by side, plus word error rate and timing.

Targets from key release to text pasted:
- short sentence: under 1.5 s
- 30 s paragraph: under 3 s

## Risks

- Whisper may spell Tagalog phonetically or translate it into English. Mitigate with the initial prompt, language setting, and the Claude cleanup step.
- Some apps running as admin block simulated input from a non-admin app. Document it; don't run Salita as admin by default.
- Antivirus may flag a PyInstaller exe with a keyboard hook. Code-signing is out of scope for now; document it.
- Privacy: transcripts (text) go to Anthropic. Audio never leaves the PC.
