# hello-world
my first repository (sort of)


I am a large land fish, fond of marshmallows and mushrooms.

## record-prompt

`bin/record-prompt` — voice input for Claude Code that keeps rambling
transcripts out of the primary model's context. Records from the mic,
transcribes locally with whisper.cpp, condenses the transcript with Claude
Haiku, and prints only the tight final prompt to stdout, so `! record-prompt`
inside a Claude Code session hands the model a token-efficient prompt.

Setup and details: [docs/voice-prompt.md](docs/voice-prompt.md)
