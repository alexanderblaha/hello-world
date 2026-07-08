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

### 2. Whisper model (one-time, ~150 MB)

```sh
mkdir -p ~/.local/share/whisper
curl -L -o ~/.local/share/whisper/ggml-base.en.bin \
  https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.en.bin
```

`base.en` transcribes a 60-second clip in ~1–2 s on a modern laptop and is
accurate enough for prompt dictation. Use `ggml-small.en.bin` if you dictate
a lot of jargon (set `RECORD_PROMPT_WHISPER_MODEL` to its path).

### 3. Credentials — none needed with a Claude subscription

By default the script condenses through the `claude` CLI in print mode
(`claude -p --model claude-haiku-4-5 ...`), which runs on your **Claude
Pro/Max subscription** — if you're already logged in to Claude Code, there is
nothing to configure and no API billing.

To use the raw Anthropic API instead (pay-per-token):

```sh
export RECORD_PROMPT_CONDENSER=api
export ANTHROPIC_API_KEY=sk-ant-...
```

(`ANTHROPIC_AUTH_TOKEN` from `ant auth login` also works for the api backend.)

### 4. Put it on your PATH

```sh
ln -s "$(pwd)/bin/record-prompt" ~/.local/bin/record-prompt
```

Or just invoke it by path from Claude Code: `! bin/record-prompt`.

## Usage

| Command | Effect |
|---|---|
| `record-prompt` | record → transcribe → condense → stdout |
| `record-prompt --raw` | print the raw transcript (skip Haiku) — for debugging |
| `record-prompt --text "..."` | condense given text (no mic needed) — for testing |
| `echo "..." \| record-prompt --text -` | same, from stdin |
| `record-prompt --keep-audio` | keep the temporary `.wav` |

## Configuration

| Env var | Default | Purpose |
|---|---|---|
| `RECORD_PROMPT_CONDENSER` | auto (`claude-cli` if the `claude` binary is found, else `api`) | force `claude-cli` (subscription) or `api` (API key) |
| `RECORD_PROMPT_MODEL` | `claude-haiku-4-5` | condenser model |
| `RECORD_PROMPT_WHISPER_MODEL` | `~/.local/share/whisper/ggml-base.en.bin` | whisper.cpp model path |
| `RECORD_PROMPT_TRANSCRIBER` | auto (`local` if whisper found, else `openai`) | force `local` or `openai` |
| `OPENAI_API_KEY` | — | enables the OpenAI audio-API fallback |
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
