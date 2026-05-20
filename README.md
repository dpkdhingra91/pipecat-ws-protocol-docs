# Pipecat WebSocket Protocol — Reference

Independent reference documentation for the Pipecat WebSocket transport, written for client implementers (Python, Go, Rust, anywhere you don't have a JS SDK).

> **Why this exists:** as of writing, the Pipecat WebSocket protocol is fully implemented in `@pipecat-ai/client-js` (browser) but undocumented outside the source. The Python `pipecat-ai` package on PyPI is the server side. If you want to connect *to* a Pipecat server from any language other than JavaScript, you have to read the source. This document captures what I learned doing exactly that. — Deepak

Companion library: [`voice-agent-qa`](https://github.com/dpkdhingra91/voice-agent-qa) — Python implementation of this protocol.

---

## 1. The two-step handshake

The browser (and any other client) does a two-step handshake:

### Step 1: `POST /connect`

```
POST https://<your-pipecat-host>/connect
Content-Type: application/json

{
  "meeting_id": "<uuid>",
  ...whatever app-specific fields your server's /connect route expects
}
```

Returns:
```json
{ "ws_url": "/ws?sid=<12-hex-id>" }
```

The `sid` is a **one-shot redemption coupon**. The server pops it from its in-memory `_sessions` dict the moment the WebSocket connects. You have ~10 seconds to open the WS before the server times the session out.

### Step 2: open the WebSocket

```
WS  wss://<your-pipecat-host><ws_url>
```

Note: the response's `ws_url` is typically a relative path. You concatenate it with your `wss://` base.

---

## 2. Wire format — Protobuf, not JSON

Every WebSocket message is a binary serialized [`pipecat.frames.frames.Frame`](https://github.com/pipecat-ai/pipecat/blob/main/src/pipecat/frames/frames.proto) protobuf:

```proto
syntax = "proto3";
package pipecat;

message TextFrame {
  uint64 id = 1;
  string name = 2;
  string text = 3;
}

message AudioRawFrame {
  uint64 id = 1;
  string name = 2;
  bytes audio = 3;
  uint32 sample_rate = 4;
  uint32 num_channels = 5;
  optional uint64 pts = 6;
}

message TranscriptionFrame {
  uint64 id = 1;
  string name = 2;
  string text = 3;
  string user_id = 4;
  string timestamp = 5;
}

message MessageFrame {
  string data = 1;
}

message Frame {
  oneof frame {
    TextFrame text = 1;
    AudioRawFrame audio = 2;
    TranscriptionFrame transcription = 3;
    MessageFrame message = 4;
  }
}
```

The server's serializer config:
```python
ProtobufFrameSerializer()
audio_in_enabled = True,   audio_in_sample_rate  = 16000
audio_out_enabled = True,  audio_out_sample_rate = 24000
add_wav_header = False
```

---

## 3. RTVI envelopes ride inside `MessageFrame.data`

Every "event" is JSON wrapped in an RTVI envelope, transported as the string payload of a `MessageFrame`:

```json
{
  "label": "rtvi-ai",
  "type":  "<event-type>",
  "id":    "<message-uuid>",
  "data":  { ...event-specific... }
}
```

RTVI is a small protocol (version 1.3.0 at time of writing). The event types you need to handle:

### Server → client

| Event type | Data shape | Notes |
|---|---|---|
| `bot-ready` | `{ "version": "1.3.0", "about": {...} }` | Server says "I'm initialized" |
| `bot-transcription` | `{ "text": "..." }` | Full assistant turn text |
| `user-transcription` | `{ "text": "...", "user_id": "", "timestamp": "", "final": bool }` | STT result, interim or final |
| `bot-started-speaking` | (no data) | TTS begins streaming audio |
| `bot-stopped-speaking` | (no data) | TTS stream complete |
| `server-message` | `{ "type": "<app-specific>", ... }` | App-defined server-side events |
| `error` | `{ "error": "...", "fatal": bool }` | Server-side pipeline error |

### Client → server

| Event type | Data shape | Notes |
|---|---|---|
| `client-ready` | `{ "version": "1.3.0", "about": {"library": "your-lib"} }` | Send immediately after WS opens. Triggers the server's initial `LLMRunFrame`. |
| `client-message` | `{ "t": "<type>", "d": {...} }` | App-defined client→server messages |
| `disconnect-bot` | (no data) | Polite shutdown — triggers `EndTaskFrame` in the server pipeline |

The JS SDK re-flattens these for app code: `client.on("botTranscript", data)` is the RTVI envelope's `data` field for a `bot-transcription` event.

---

## 4. Audio frame shape

### Client → server (mic input)

- Format: 16 kHz mono PCM, 16-bit little-endian
- Chunk size: 512 samples per frame (1024 bytes)
- Carried in `AudioRawFrame.audio`

### Server → client (TTS output)

- Format: 24 kHz mono PCM, no WAV header
- Variable chunk size
- Carried in `AudioRawFrame.audio`

The server expects mic chunks to be pre-padded/truncated to the fixed size by the caller — the wire protocol doesn't enforce it, but mis-sized chunks will be regrouped and may cause STT artifacts.

---

## 5. Common implementation pitfalls

### Don't `await ws.send()` before the server sends `bot-ready`

Some servers buffer client messages before the pipeline is constructed. Send `client-ready` immediately, but wait for `bot-ready` before sending audio.

### Send `client-ready` even if you don't need to

The server's `RTVIProcessor` waits for it to trigger the initial pipeline run. Without it, the bot will never start its first turn.

### Don't reuse the `sid` from POST /connect

It's one-shot. If the WS handshake fails (network blip), you must POST /connect again to get a fresh sid. Don't retry the WS connection with the same URL.

### Binary messages only, except…

If you receive a text WebSocket frame, the server probably crashed and is sending a JSON error response. Log it and tear down.

### Mind the `pts` field on `AudioRawFrame`

Optional — most clients ignore it. If present (server side), it's a presentation timestamp in microseconds. Helpful for synchronizing audio output but not required for playback.

---

## 6. Reverse-engineering notes (history)

I figured this out by reading:

1. [`pipecat/transports/network/websocket_server.py`](https://github.com/pipecat-ai/pipecat/blob/main/src/pipecat/transports/network/websocket_server.py) — server WS handler
2. [`pipecat/serializers/protobuf.py`](https://github.com/pipecat-ai/pipecat/blob/main/src/pipecat/serializers/protobuf.py) — wire format
3. [`pipecat/processors/frameworks/rtvi/`](https://github.com/pipecat-ai/pipecat/tree/main/src/pipecat/processors/frameworks/rtvi) — RTVI envelope handlers
4. [`@pipecat-ai/client-js`](https://github.com/pipecat-ai/pipecat-client-web) — browser SDK reference implementation

If you're implementing this in another language, those four are the source of truth. This document is a faithful summary as of Pipecat 0.0.50 / RTVI 1.3.0 — if upstream evolves, treat this as historical.

---

## 7. Working Python reference

See [`voice-agent-qa`](https://github.com/dpkdhingra91/voice-agent-qa) for a complete, tested Python client implementing this protocol. ~300 lines, MIT, no `torch` / `transformers` / `onnxruntime`.

## Related projects

- 🎯 [`pipecat-sarvam-azure-starter`](https://github.com/dpkdhingra91/pipecat-sarvam-azure-starter) — a complete Pipecat server (Sarvam STT + Azure OpenAI + Sarvam TTS) that speaks the protocol documented here.
- 🐍 [`voice-agent-qa`](https://github.com/dpkdhingra91/voice-agent-qa) — Python client that implements this protocol; useful as a working reference if you're porting to another language.

## License

This documentation is licensed CC0 — public domain. Copy, paste, translate, mirror, embed, change, do whatever.
