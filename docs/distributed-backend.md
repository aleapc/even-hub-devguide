# Distributed Backend Pattern

When your G2 app needs capabilities that don't fit in a WebView — heavy computation, ML inference, external API calls, persistent databases, real-time data from a server — offload them to a backend service.

This pattern is used by [claude-code-g2](https://github.com/sam-siavoshian/claude-code-g2), [openclaw-g2-hud](https://github.com/kqb/openclaw-g2-hud), and our own [speechcoach-backend](https://github.com/aleapc/speechcoach-backend).

## Architecture

```
[G2 glasses] ←BLE→ [Phone WebView (your G2 app)] ←HTTPS/SSE→ [Backend server]
                                                                      ↓
                                                            [External APIs / ML / DB]
```

Three layers:

1. **G2 app (WebView)** — thin client. Renders the glasses UI, captures input, relays audio.
2. **Backend server** — does the heavy work. Node/Python/Rust, wherever you're comfortable.
3. **External dependencies** — APIs, ML models, databases, message queues.

## When to use this pattern

✅ **Use it when:**
- You need speech-to-text, translation, or LLM inference (too heavy for WebView)
- You need a persistent database or queryable history
- You need to integrate with external services (Slack, Notion, email, webhooks)
- Multiple users share state (collaborative features)
- You need real-time data from a server (stock prices, game state, chat)

❌ **Don't use it when:**
- Your app works entirely offline (keep it in the WebView)
- Your data fits comfortably in `bridge.setLocalStorage` (a few KB of settings/stats)
- You're prototyping or doing something simple (the extra moving parts slow you down)

## Communication: SSE is usually the right choice

Three options from simplest to heaviest:

| Pattern | Pros | Cons | When |
|---|---|---|---|
| **HTTP POST (one-shot)** | Simplest, stateless | No live updates | Infrequent requests |
| **Server-Sent Events (SSE)** | Simple, one-way streaming, auto-reconnects | Server → client only | Live updates from backend |
| **WebSocket** | Full bidirectional | More complexity, auth tricky | Bidirectional real-time |

For most G2 app use cases, **HTTP POST for actions + SSE for live updates** is the right split:

- Glass app does `POST /session/start` → backend returns session ID
- Glass app opens `GET /session/:id/stream` as SSE
- Backend pushes events to the stream (transcription updates, notifications, state changes)
- Glass app does `POST /session/:id/end` when the user finishes

This is how [speechcoach-backend](https://github.com/aleapc/speechcoach-backend) works.

## Minimal Express backend skeleton

```typescript
// src/server.ts
import express from 'express'
import cors from 'cors'

const app = express()
app.use(cors())
app.use(express.json({ limit: '10mb' }))

// Health check
app.get('/health', (_req, res) => res.json({ ok: true }))

// One-shot action
app.post('/sessions', (req, res) => {
  const sessionId = crypto.randomUUID()
  // Create session in your store...
  res.json({ sessionId })
})

// SSE stream
app.get('/sessions/:id/stream', (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
  })

  const sessionId = req.params.id
  const subscriber = (data: unknown) => {
    res.write(`data: ${JSON.stringify(data)}\n\n`)
  }

  subscribeToSession(sessionId, subscriber)

  req.on('close', () => {
    unsubscribeFromSession(sessionId, subscriber)
  })
})

// Action that triggers events on the stream
app.post('/sessions/:id/audio', async (req, res) => {
  const { sessionId } = req.params
  const audioBytes = req.body.pcm as number[]
  // Process... then broadcast to subscribers
  broadcastToSession(sessionId, { type: 'transcription', text: '...' })
  res.json({ ok: true })
})

app.listen(8787, () => console.log('Backend on :8787'))
```

## G2 app SSE client

```typescript
// src/glasses/backend.ts
export class BackendClient {
  private baseUrl: string
  private sessionId: string | null = null
  private eventSource: EventSource | null = null

  constructor(baseUrl: string) {
    this.baseUrl = baseUrl
  }

  async createSession(): Promise<void> {
    const res = await fetch(`${this.baseUrl}/sessions`, { method: 'POST' })
    const data = await res.json()
    this.sessionId = data.sessionId
  }

  connectStream(onEvent: (data: unknown) => void): () => void {
    if (!this.sessionId) throw new Error('No session')
    this.eventSource = new EventSource(`${this.baseUrl}/sessions/${this.sessionId}/stream`)
    this.eventSource.onmessage = (e) => {
      try { onEvent(JSON.parse(e.data)) } catch {}
    }
    this.eventSource.onerror = (e) => {
      console.warn('SSE error', e)
      // EventSource auto-reconnects
    }
    return () => this.eventSource?.close()
  }

  async sendAudio(pcm: number[]): Promise<void> {
    if (!this.sessionId) return
    await fetch(`${this.baseUrl}/sessions/${this.sessionId}/audio`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ pcm }),
    })
  }
}
```

## Configuration & user settings

The G2 app needs to know the backend URL. Options:

1. **Hardcoded URL** — simplest but not flexible
2. **User-configurable via phone UI** — best UX, stored via `bridge.setLocalStorage`
3. **Environment variable via Vite** — `VITE_BACKEND_URL` at build time

We use option 2 in `speechcoach-g2`:

```typescript
// Phone UI (React) has a text input
const [backendUrl, setBackendUrl] = useState('http://localhost:8787')

useEffect(() => {
  bridge.getLocalStorage('backend_url').then(url => {
    if (url) setBackendUrl(url)
  })
}, [])

const saveUrl = async (url: string) => {
  await bridge.setLocalStorage('backend_url', url)
  setBackendUrl(url)
}
```

## Auth

If your backend exposes sensitive data, add auth. For local dev, a simple shared token via header is enough:

```typescript
// Client
fetch(`${url}/endpoint`, {
  headers: { 'Authorization': `Bearer ${token}` },
})

// Server
app.use((req, res, next) => {
  const auth = req.headers.authorization
  if (auth !== `Bearer ${process.env.AUTH_TOKEN}`) {
    return res.status(401).json({ error: 'unauthorized' })
  }
  next()
})
```

For production multi-user apps, use proper auth (JWT, OAuth, etc).

## Deployment

Backends need to be reachable from the phone's WebView. Options:

- **Local dev**: `localhost:8787` — works if you're testing on your own phone on the same Wi-Fi as your laptop. Use your LAN IP in the backend URL setting.
- **ngrok / Cloudflare Tunnel**: expose your local backend over HTTPS for testing with other users
- **Cloudflare Workers**: for light backends (our `g2-telemetry-worker` uses this)
- **Fly.io / Railway / Render**: for heavier Node/Python backends
- **Self-hosted VPS**: for full control

Whichever you choose, make sure CORS allows `null` origin (some WebViews send `null`) or use a wildcard for development.

## Example: speechcoach-backend

See [`speechcoach-backend`](https://github.com/aleapc/speechcoach-backend) for a complete example:

- `src/server.ts` — Express server with `/sessions` + SSE streams
- `src/transcribe.ts` — Pluggable STT (OpenAI Whisper / Deepgram / mock)
- `src/analysis.ts` — Speech metrics (WPM, fillers, pauses)
- `src/sessions.ts` — In-memory session store with pub/sub

And the G2-side [`speechcoach-g2`](https://github.com/aleapc/speechcoach-g2) `src/glasses/backend.ts` shows the client integration with graceful fallback to local-only mode when the backend is unreachable.

## Credits

The SSE pattern for G2 apps was popularized by:
- [sam-siavoshian/claude-code-g2](https://github.com/sam-siavoshian/claude-code-g2) — distributed architecture with `claude` CLI backend
- [kqb/openclaw-g2-hud](https://github.com/kqb/openclaw-g2-hud) — production WebSocket pattern with auto-reconnect
