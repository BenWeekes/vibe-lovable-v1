# Agora Conversational AI Voice Agent

A real-time voice AI agent powered by [Agora Conversational AI](https://www.agora.io/en/products/conversational-ai/), [OpenAI](https://platform.openai.com/) LLM, and your choice of TTS provider. Built with Vite + React + TypeScript + Supabase Edge Functions. Click **Start Call**, talk, and see live transcripts in the chat panel.

## Deploy with an AI Coding Platform

Use this prompt to import the repo into your AI coding platform:

> Build this Agora Voice AI Agent: https://github.com/BenWeekes/vibe-lovable-v1 — be sure to read AGENT.md in full.

When prompted for environment variables, provide:

| Variable | Required | Description |
|----------|----------|-------------|
| `APP_ID` | Yes | Agora App ID ([Agora Console](https://console.agora.io)) |
| `APP_CERTIFICATE` | Yes | Agora App Certificate (32-char hex) — used with `APP_ID` to generate v007 tokens inline for RTC/RTM and API auth |
| `LLM_API_KEY` | Yes | LLM provider API key (e.g. OpenAI) |
| `TTS_VENDOR` | Yes | `rime`, `openai`, `elevenlabs`, or `cartesia` |
| `TTS_KEY` | Yes | TTS provider API key |
| `TTS_VOICE_ID` | Yes | Voice ID (e.g. `astra` for Rime, `alloy` for OpenAI) |
| `LLM_URL` | No | LLM endpoint (default: OpenAI) |
| `LLM_MODEL` | No | Model name (default: `gpt-4o-mini`) |

### Platform-Specific Tips

<!-- PLATFORM_TIPS_START -->

**Lovable:**
- Secrets are set via the Supabase dashboard (linked from Lovable project settings)
- The app uses Supabase Edge Functions instead of a traditional backend
- Lovable auto-links a Supabase project — set the secrets there

**v0 (Vercel):**
- For the Next.js variant, see [vibe-code-v0](https://github.com/AgoraIO-Conversational-AI/vibe-code-v0)

<!-- PLATFORM_TIPS_END -->

## Run Locally

### Prerequisites

- Node.js 20+ (`nvm use 20`)
- Agora credentials, LLM API key, and TTS credentials (see table above)

### Setup

```bash
npm install
```

### 1. Configure `.env`

Point the Supabase client at the local test server:

```
VITE_SUPABASE_PROJECT_ID="local"
VITE_SUPABASE_PUBLISHABLE_KEY="local"
VITE_SUPABASE_URL="http://localhost:3002"
```

### 2. Start the test server

`test-server.mjs` is included in the repo — it's a Node.js server that mimics Supabase Edge Functions locally.

```bash
APP_ID=<your_app_id> \
APP_CERTIFICATE=<your_app_certificate> \
LLM_API_KEY=<your_llm_key> \
TTS_VENDOR=rime \
TTS_KEY=<your_tts_key> \
TTS_VOICE_ID=astra \
node test-server.mjs
```

Starts on port 3002.

### 3. Start the Vite dev server (separate terminal)

```bash
npm run dev
```

Open http://localhost:8080

### How It Works

Since Supabase Edge Functions (Deno) can't easily run locally, `test-server.mjs` is a Node.js server that mimics the edge function endpoints:

```
Browser (Vite, port 8080)
  └── VITE_SUPABASE_URL → test-server (port 3002)
        ├── /functions/v1/check-env
        ├── /functions/v1/start-agent
        └── /functions/v1/hangup-agent
```

Credentials are passed as environment variables to the test server, not stored in files.

### Troubleshooting

- **"Configuration Required"** — The test server is missing credentials. Check it printed all required vars on startup.
- **`crypto.getRandomValues` error** — You need Node.js 20+. Run `nvm use 20`.
- **Port in use** — `lsof -ti:3002 | xargs kill` or `lsof -ti:8080 | xargs kill`

## Features

- **Real-time Voice** — Full-duplex audio via Agora RTC with echo cancellation, noise suppression, and auto gain control
- **Live Transcripts** — User and agent speech appears in the chat window as it happens
- **Text Chat** — Type a message and send it to the agent via Agora RTM
- **Agent Visualizer** — Animated orb (idle, joining, listening, speaking, disconnected)
- **Customizable** — Settings panel for system prompt and greeting
- **Self-contained** — Supabase Edge Functions handle token generation, agent start, and hangup

## Architecture

```
Browser (React + Agora RTC/RTM SDK)
  │
  ├── Supabase Edge Function: start-agent
  │     └── Agora ConvoAI API
  │
  ├── Agora RTC: audio (UID 100)
  │
  └── Agora RTM: text messaging
```

- **UID 100** — Agent audio
- **UID 101** — User

## Tech Stack

- **Framework:** Vite + React 18
- **Language:** TypeScript 5
- **Styling:** Tailwind CSS + shadcn/ui
- **Backend:** Supabase Edge Functions (Deno)
- **RTC SDK:** agora-rtc-sdk-ng v4.24+
- **RTM SDK:** agora-rtm v2.2+
- **Token gen:** v007 token builder (inline, server-side)

## License

MIT
