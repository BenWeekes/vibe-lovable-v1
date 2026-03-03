# Agent Instructions

This file is for AI coding assistants. Read it in full before making changes.

## Project Overview

A Vite + React 18 + TypeScript app (React Router, Tailwind CSS, shadcn/ui) that connects to an Agora Conversational AI voice agent. The app handles real-time audio via Agora RTC, text messaging via Agora RTM, and live transcripts via RTC stream-message. Server-side logic runs in Supabase Edge Functions (Deno).

**This code has been tested locally and is built with the Lovable platform in mind.** All files are complete and working — import them as-is and wire up environment variables. Do not rewrite or restructure existing components.

## Critical Rules

- **Do NOT modify `src/index.css`** — it contains the custom Agora brand theme (dark + light mode with cyan/teal primary colors, agent orb CSS vars, scrollbar styles).
- **Do NOT modify files in `src/components/ui/`** — these are stock shadcn/ui components.
- **Do NOT inline Agora SDK imports at file top level** — `agora-rtc-sdk-ng` and `agora-rtm` require browser APIs and crash SSR. They must be dynamically imported inside async functions.
- **Do NOT add transcripts to the chat UI on RTM send** — the agent echoes them back via the RTC stream-message listener, so they appear automatically.
## Styling Rules

- All primary action buttons (mic toggle when unmuted, chat send button) use `bg-primary text-primary-foreground` — they must match visually
- Destructive actions (hangup button) use `bg-destructive text-destructive-foreground`
- Muted/disabled state (mic when muted) uses `bg-muted text-muted-foreground`
- The theme is defined in `src/index.css` via CSS custom properties — do not hardcode colors, use the Tailwind theme tokens (`primary`, `muted`, `destructive`, etc.)

## File Structure

```
src/
  index.css              — Custom Agora theme (DO NOT MODIFY)
  App.tsx                — Root app with router, providers
  main.tsx               — Entry point
  pages/
    Index.tsx            — Renders VoiceClient
    NotFound.tsx         — 404 page
  components/
    VoiceClient.tsx      — Main UI component (525 lines) — orb, chat, settings, controls
    AgoraLogo.tsx        — Agora SVG logo
    ThemeProvider.tsx     — Dark/light theme context
    ThemeToggle.tsx       — Theme toggle button
    NavLink.tsx          — Navigation link
  components/ui/         — Stock shadcn/ui (do not modify)
  hooks/
    useAgoraVoiceClient.ts — All Agora SDK logic, state, refs (the core hook, 298 lines)
    useAudioVisualization.ts — Web Audio API AnalyserNode for waveform
    use-toast.ts         — Toast notifications
    use-mobile.tsx       — Mobile detection
  integrations/supabase/
    client.ts            — Supabase client init
    types.ts             — Supabase types
  lib/
    utils.ts             — cn() utility

supabase/functions/
  check-env/index.ts     — Validates required env vars (47 lines)
  start-agent/index.ts   — Generates tokens, starts ConvoAI agent (333 lines)
  hangup-agent/index.ts  — Stops the agent (156 lines)
  health/index.ts        — Health check (15 lines)

test-server.mjs          — Local Node.js server mimicking Supabase Edge Functions
```

## Environment Variables

### Supabase Secrets (server-side, set via `npx supabase secrets set`)

Required:
- `APP_ID` — Agora App ID
- `APP_CERTIFICATE` — Agora App Certificate (32-char hex). Used with APP_ID to generate v007 tokens inline for both RTC/RTM access and Agora Conversational AI API auth — no separate Customer Key/Secret or npm token package needed
- `LLM_API_KEY` — LLM provider API key (e.g. OpenAI)
- `TTS_VENDOR` — `rime`, `openai`, `elevenlabs`, or `cartesia`
- `TTS_KEY` — TTS provider API key
- `TTS_VOICE_ID` — Voice ID (e.g. `astra` for Rime, `alloy` for OpenAI)

Optional:
- `LLM_URL` — Default: `https://api.openai.com/v1/chat/completions`
- `LLM_MODEL` — Default: `gpt-4o-mini`

### Vite Environment (`.env` in project root)

```
VITE_SUPABASE_PROJECT_ID="<your_project_id>"
VITE_SUPABASE_PUBLISHABLE_KEY="<your_anon_key>"
VITE_SUPABASE_URL="https://<your_project_id>.supabase.co"
```

## Supabase Edge Function: `POST /functions/v1/start-agent`

Accepts optional POST body `{ prompt, greeting }`. Defaults: prompt = "You are a friendly voice assistant. Keep responses concise, around 10 to 20 words." greeting = "Hi there! How can I help you today?"

**Token generation** — v007 token builder (inline, no npm package) that creates combined RTC+RTM tokens with separate UIDs. The RTC service uses the channel uid (e.g. `"100"`) while the RTM service uses a distinct RTM uid (e.g. `"100-{channel}"`).

UIDs are strings: agent = `"100"`, user = `"101"`. Channel is random 10-char alphanumeric. Agent RTM UID = `"100-{channel}"`.

**Agent payload** — POST to `https://api.agora.io/api/conversational-ai-agent/v2/projects/{appId}/join`:

```json
{
  "name": "{channel}",
  "properties": {
    "channel": "{channel}",
    "token": "{agentToken}",
    "agent_rtc_uid": "100",
    "agent_rtm_uid": "100-{channel}",
    "remote_rtc_uids": ["*"],
    "enable_string_uid": false,
    "idle_timeout": 120,
    "advanced_features": {
      "enable_bhvs": true,
      "enable_rtm": true,
      "enable_aivad": true,
      "enable_sal": false
    },
    "llm": {
      "url": "{LLM_URL or https://api.openai.com/v1/chat/completions}",
      "api_key": "{LLM_API_KEY}",
      "system_messages": [{ "role": "system", "content": "{prompt}" }],
      "greeting_message": "{greeting}",
      "failure_message": "Sorry, something went wrong",
      "max_history": 32,
      "params": { "model": "{LLM_MODEL or gpt-4o-mini}" },
      "style": "openai"
    },
    "vad": { "silence_duration_ms": 300 },
    "asr": { "vendor": "ares", "language": "en-US" },
    "tts": "{ttsConfig}",
    "parameters": {
      "transcript": {
        "enable": true,
        "protocol_version": "v2",
        "enable_words": false
      }
    }
  }
}
```

**TTS config builder** — supports multiple vendors:

- **rime** (default): `{ vendor: "rime", params: { api_key, speaker: voiceId, modelId: "mistv2", lang: "eng", samplingRate: 16000, speedAlpha: 1.0 } }`
- **openai**: `{ vendor: "openai", params: { api_key, model: "tts-1", voice: voiceId, response_format: "pcm", speed: 1.0 } }`
- **elevenlabs**: `{ vendor: "elevenlabs", params: { key, model_id: "eleven_flash_v2_5", voice_id: voiceId, stability: 0.5, sample_rate: 24000 } }`
- **cartesia**: `{ vendor: "cartesia", params: { api_key, model_id: "sonic-3", sample_rate: 24000, voice: { mode: "id", id: voiceId } } }`

Returns: `{ appId, channel, token, uid, agentUid, agentRtmUid, agentId, success }`

## Supabase Edge Function: `POST /functions/v1/hangup-agent`

POST with `{ agentId }`. Calls `POST https://api.agora.io/api/conversational-ai-agent/v2/projects/{appId}/agents/{agentId}/leave` with token-based auth header.

## Frontend: Browser-Only SDK Imports

Both `agora-rtc-sdk-ng` and `agora-rtm` require browser APIs. They must be dynamically imported inside async functions, never at the top of the file:

```typescript
const AgoraRTC = (await import("agora-rtc-sdk-ng")).default;
const { default: AgoraRTM } = await import("agora-rtm");
```

## Frontend: RTC Voice + Transcript Listener

**Register ALL event listeners BEFORE `client.join()`.** The `stream-message` listener is critical — it receives ALL transcripts (both user speech and agent responses).

**IMPORTANT: Transcripts arrive via RTC `stream-message`, NOT via RTM.** Protocol v2 encodes transcripts as base64 inside a pipe-delimited string (`messageId|partIdx|partSum|base64data`) — you MUST split on `|`, accumulate chunks by messageId, `atob()` the joined base64, then `JSON.parse()`. Raw `JSON.parse()` on the stream data will NOT work. Both user speech transcripts and agent response transcripts come through this single listener. The agent greeting also arrives here — do not hardcode it. Display transcripts as chat bubbles grouped by `turn_id`. Update in-place for partial transcripts, mark final when complete.

## Frontend: RTM Text Messaging (send only)

RTM is used **ONLY for sending text messages** from the user to the agent. Do NOT use `createStreamChannel`, `joinTopic`, `publishTopicMessage`, or `sendMessage`.

```typescript
const payload = JSON.stringify({ message: text, priority: "APPEND" });
await rtm.publish(agentRtmUid, payload, {
  customType: "user.transcription",
  channelType: "USER",
});
```

**IMPORTANT RTM rules:**

- Publish target is `agentRtmUid` (e.g. `"100-{channel}"`), NOT the channel name
- Message must be JSON: `{ "message": "text", "priority": "APPEND" }`
- Options must include `customType: "user.transcription"` and `channelType: "USER"`
- Do NOT add the message to the chat UI locally — the agent echoes it back as a `user.transcription` via the RTC stream-message transcript listener, so it appears automatically
- Never `console.log()` the RTM client object — it causes `RangeError: Invalid string length` from circular references

## Frontend: UI Layout

**Pre-connection (idle state):** Left panel with centered orb, Connect button, error display. Right panel with placeholder text. Settings button in header opens modal for system prompt and greeting (disabled while connected).

**Connected:** Left panel has animated orb (pulsing when listening, scaling + glowing when speaking, spinning border when joining), waveform bars from Web Audio API AnalyserNode, mute/unmute + hangup buttons. Right panel has scrolling chat with agent/user bubbles (in-progress messages show bouncing dots and reduced opacity), text input with send button (only visible when connected). Header shows app title, elapsed timer, "Live" indicator, settings gear.

**Agent orb states:** idle (dim, scaled down), joining (spinning border), listening (gentle pulse), talking (ping rings + glow + scale up), disconnected (dim).
