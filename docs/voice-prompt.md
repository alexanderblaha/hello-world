# `record-prompt` — token-optimized voice input for Claude Code

Speak a rambling request; hand Claude Code a tight, token-efficient prompt.

```
  mic ──▶ recorder ──▶ whisper.cpp ──▶ Claude Haiku ──▶ stdout ──▶ ! bash mode
         (sox/ffmpeg)  (local STT)    (condensation)              (Fable's context)
```

From inside a Claude Code session:

```
! record-prompt
```

A `● Recording...` indicator appears; talk, press **Enter** to stop. The script
transcribes the audio, condenses it with Claude Haiku (`claude-haiku-4-5`,
$1/$5 per MTok), and prints **only the condensed prompt** to stdout — which is
exactly what Claude Code's `!` mode inserts into the primary model's context.

## What this costs

- **whisper.cpp: free.** Open source (MIT), model weights are a free download,
  runs entirely on your machine — no account, no subscription, no per-use cost.
- **Haiku condensation: covered by your Claude Pro/Max subscription** via the
  `claude` CLI (the default). With the `api` backend it's pay-per-token
  instead (`claude-haiku-4-5` at $1/$5 per MTok — a fraction of a cent per
  recording).
- **OpenAI transcription fallback (optional, off by default):**
  ~$0.003–0.006/min. Never needed if whisper.cpp is installed.

## Why this saves real money

The savings compound. Claude Code resends the full conversation on every
API call, so a 400-token raw transcript isn't paid once — it's paid on **every
subsequent turn** of the session. Condensing it to ~60 tokens up front saves
`(400 − 60) × remaining_turns` input tokens on the expensive primary model,
for a fraction of a cent of Haiku usage per recording.

## The `/dev/tty` trick (important)

Claude Code's `!` mode captures **both stdout and stderr** into the model's
context. If the recording UI wrote to stderr, the `● Recording...` chrome
would leak tokens into the primary model too. So all interactive output goes
to `/dev/tty` (the terminal directly), and the *only* bytes on stdout are the
final Haiku output. Keep this invariant if you modify the script.

## Setup

### 1. Dependencies

macOS:

```sh
brew install sox whisper-cpp jq
```

Debian/Ubuntu:

```sh
sudo apt install sox jq        # or: alsa-utils for arecord
# whisper.cpp: build from source or use your package manager
# https://github.com/ggerganov/whisper.cpp
```

Windows — two options:

**Option A (recommended): WSL2.** Run Claude Code and this script inside WSL;
everything behaves like the Linux instructions above (`sudo apt install sox jq`,
whisper.cpp from source or `brew install whisper-cpp` via Homebrew-on-Linux).
Microphone access inside WSL2 comes through WSLg's PulseAudio passthrough
(Windows 11, or Windows 10 with WSLg installed) — verify with `pactl info`;
if a mic shows up, `sox`/`ffmpeg -f pulse` recording works.

**Running Claude Code from PowerShell (VS Code terminal) or the desktop app?**
You're on Option B without realizing it: Claude Code on native Windows executes
`!` commands through Git Bash regardless of which shell hosts it, so
`! bin/record-prompt` works from a PowerShell-hosted session once the Option B
dependencies below are installed. Embedded terminals don't always give the
script an interactive keyboard for a "stop" keypress — in that case it uses
**silence auto-stop**: just talk, then stay quiet for ~2 seconds and the
recording ends on its own. That needs sox installed (`scoop install sox` or
`choco install sox.portable`); see "Stopping the recording" below for all
three stop modes.

**Option B: native Git Bash.** Install the dependencies:

```sh
winget install Gyan.FFmpeg jqlang.jq
```

For whisper.cpp, download the Windows binary zip from the
[whisper.cpp releases page](https://github.com/ggerganov/whisper.cpp/releases)
and put `whisper-cli.exe` somewhere on your PATH. The script auto-detects
Windows and records via ffmpeg's DirectShow capture — the recording prompt
says **press `q` to stop** (instead of Enter), because that's how ffmpeg stops
cleanly there. Your default microphone is auto-detected; to pick a specific
one, list devices and set it:

```sh
ffmpeg -list_devices true -f dshow -i dummy   # shows "(audio)" device names
export RECORD_PROMPT_MIC="Microphone (Realtek(R) Audio)"
```

### 2. Whisper model (one-time, ~150 MB)

```sh
mkdir -p ~/.local/share/whisper
curl -L -o ~/.local/share/whisper/ggml-base.en.bin \
  https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.en.bin
```

`base.en` transcribes a 60-second clip in ~1–2 s on a modern laptop and is
accurate enough for prompt dictation. Use `ggml-small.en.bin` if you dictate
a lot of jargon (set `RECORD_PROMPT_WHISPER_MODEL` to its path).

### 3. Credentials — subscriptions cover condensation, not transcription

Four condenser backends are supported; the two CLI backends bill against
subscriptions you may already have, the two API backends are pay-per-token:

| Backend | Runs on | Billing |
|---|---|---|
| `claude-cli` (default) | `claude -p` | **Claude Pro/Max subscription** — no key needed |
| `codex` | `codex exec` | **ChatGPT subscription** — no key needed (`npm i -g @openai/codex && codex login`) |
| `api` | Anthropic Messages API | pay-per-token, needs `ANTHROPIC_API_KEY` (or `ANTHROPIC_AUTH_TOKEN`) |
| `openai-api` | OpenAI chat completions | pay-per-token, needs `OPENAI_API_KEY` |

If you're already logged in to Claude Code there is nothing to configure —
the default condenses via your Claude subscription.

**Note on transcription:** a ChatGPT subscription does *not* include OpenAI
API access, and OpenAI's speech-to-text has no subscription-backed CLI route —
so cloud transcription is always pay-per-minute API usage. Local whisper.cpp
(free) remains the recommended transcriber regardless of which condenser you
pick.

### 4. Put it on your PATH

```sh
ln -s "$(pwd)/bin/record-prompt" ~/.local/bin/record-prompt
```

Or just invoke it by path from Claude Code: `! bin/record-prompt`.

## Stopping the recording

Three modes, chosen automatically but always overridable:

1. **Keypress** — in a normal interactive terminal: press **Enter** (macOS/
   Linux/WSL) or **q** (native Windows/ffmpeg) to stop. The default whenever
   the script can read your keyboard.
2. **Silence auto-stop** (`--silence`) — recording begins at your first word
   and ends by itself after ~2 s of quiet (hard cap 300 s). Full control with
   zero keyboard access, so it's the automatic choice in embedded terminals
   (VS Code-hosted Claude Code, the desktop app). Requires sox. Tune with
   `RECORD_PROMPT_SILENCE` (quiet seconds), `RECORD_PROMPT_SILENCE_LEVEL`
   (threshold, default `3%` — raise it in noisy rooms), and
   `RECORD_PROMPT_MAX_SECONDS` (cap).
3. **Fixed duration** (`--seconds N`) — last resort, and the fallback only if
   sox is missing in a non-interactive terminal.

## Usage

| Command | Effect |
|---|---|
| `record-prompt` | record → transcribe → condense → stdout |
| `record-prompt --condenser codex` | condense via ChatGPT subscription (also: `claude-cli`, `api`, `openai-api`) |
| `record-prompt --model sonnet` | pick the condenser model (`haiku`/`sonnet`/`opus` aliases or any full model ID) |
| `record-prompt --transcriber openai` | force cloud STT instead of local whisper |
| `record-prompt --silence` | stop automatically after ~2s of quiet (needs sox) |
| `record-prompt --seconds 60` | record a fixed duration instead of waiting for a keypress |
| `record-prompt --compare "haiku,sonnet,opus,codex:gpt-5.5"` | run the same transcript through several condensers, labeled side by side |
| `record-prompt --raw` | print the raw transcript (skip condensing) — for debugging |
| `record-prompt --text "..."` | condense given text (no mic needed) — for testing |
| `echo "..." \| record-prompt --text -` | same, from stdin |
| `record-prompt --keep-audio` | keep the temporary `.wav` |

### Comparing condensers

`--compare` takes a comma list of `[backend:]model` specs. A bare model name
picks its backend automatically (`haiku`/`sonnet`/`opus`/`claude*` → claude,
anything else → codex/openai). Record once, judge the outputs:

```
record-prompt --compare "haiku,sonnet,opus,codex:gpt-5.5"
```

Each result prints under a `=== backend:model ===` header; a backend that
fails (not installed, not logged in) reports its error on the terminal and the
comparison continues. Run comparisons standalone rather than via `!` — unless
you *want* all variants in your Claude Code context.

## Configuration

CLI flags (`--condenser`, `--model`, `--transcriber`) override the env vars.

| Env var | Default | Purpose |
|---|---|---|
| `RECORD_PROMPT_CONDENSER` | auto: first available of `claude-cli`, `api`, `codex`, `openai-api` | condenser backend |
| `RECORD_PROMPT_MODEL` | per backend: `claude-haiku-4-5` (claude), `gpt-5-mini` (openai-api), codex's own default | condenser model |
| `RECORD_PROMPT_WHISPER_MODEL` | `~/.local/share/whisper/ggml-base.en.bin` | whisper.cpp model path |
| `RECORD_PROMPT_TRANSCRIBER` | auto (`local` if whisper found, else `openai`) | force `local` or `openai` |
| `OPENAI_API_KEY` | — | enables the OpenAI audio-API fallback |
| `RECORD_PROMPT_MIC` | auto-detected | (Windows/Git Bash only) DirectShow audio device name |
| `RECORD_PROMPT_SECONDS` | — | fixed recording duration |
| `RECORD_PROMPT_SILENCE` | `2.0` | seconds of quiet that end a `--silence` recording |
| `RECORD_PROMPT_SILENCE_LEVEL` | `3%` | amplitude below which audio counts as quiet |
| `RECORD_PROMPT_MAX_SECONDS` | `300` | hard cap on a `--silence` recording |
| `RECORD_PROMPT_OPENAI_STT_MODEL` | `gpt-4o-mini-transcribe` | OpenAI STT model |

## Design decisions

**Batch transcription, not live.** A live-transcription UI (streaming partial
text while you talk) adds a lot of complexity — a streaming STT engine, screen
redraws, partial-result reconciliation — for no benefit here: you never need to
read the raw transcript, because Haiku throws it away anyway. whisper.cpp on a
sub-minute clip finishes in a couple of seconds, so record-then-transcribe
*feels* live. If you later want visual feedback while speaking, `sox`'s built-in
VU meter already provides it (it renders to `/dev/tty` during recording).

**Local whisper.cpp over a cloud STT API.** No audio leaves your machine, no
extra per-minute cost, no network latency, and the Anthropic API itself has no
audio endpoint — transcription has to come from somewhere else regardless. The
OpenAI fallback exists for machines where installing whisper.cpp isn't
practical.

**Haiku system prompt.** The condenser is instructed to preserve all technical
identifiers (paths, error messages, flags, names) verbatim, drop filler and
self-corrections (keeping the *final* intent when you correct yourself), and
output nothing but the prompt itself. Tweak `SYSTEM_PROMPT` in
`bin/record-prompt` to taste.
