# Even Hub Developer Guide

> **A community-maintained, battle-tested guide for building apps on the Even Realities Even Hub platform for the G2 smart glasses.**

This repository exists because building for the Even Hub SDK v0.0.9 involves a surprising number of undocumented quirks, silent failures, and hardware-vs-simulator discrepancies. Everything here was learned the hard way while shipping **8 production apps** ([listed below](#example-projects)) and reading every piece of community code we could find.

If you're new to G2 development and wondering *"why doesn't my tap event fire?"* or *"why does my image container render blank?"* — this guide is for you.

## Table of contents

- [Why this guide exists](#why-this-guide-exists)
- [Quick start](#quick-start)
- [Core concepts](#core-concepts)
- [SDK quirks and workarounds](#sdk-quirks-and-workarounds)
- [Recipes](#recipes)
- [Example projects](#example-projects)
- [Community & credits](#community--credits)
- [Contributing](#contributing)

---

## Why this guide exists

The official Even Hub docs are sparse. The SDK has several known bugs that are not documented anywhere. The simulator behaves differently from real hardware in ways that silently break apps in production. Community knowledge is scattered across a handful of GitHub repos.

This guide **consolidates** that tribal knowledge into one place, with copy-pasteable code and clear explanations of *why* each workaround is needed.

**Goal**: get a new developer from zero to shipping a working G2 app in **hours**, not weeks.

## Quick start

```bash
npm create vite@latest my-g2-app -- --template vanilla-ts
cd my-g2-app
npm install @evenrealities/even_hub_sdk @evenrealities/evenhub-cli @evenrealities/evenhub-simulator
```

Minimum viable app (`src/main.ts`):

```typescript
import { waitForEvenAppBridge, TextContainerProperty, CreateStartUpPageContainer } from '@evenrealities/even_hub_sdk'

async function init() {
  const bridge = await waitForEvenAppBridge()

  const text = new TextContainerProperty({
    xPosition: 4, yPosition: 4,
    width: 568, height: 280,
    borderWidth: 0, borderColor: 0, borderRadius: 0, paddingLength: 8,
    containerID: 0, containerName: 'main',
    isEventCapture: 1,
    content: 'Hello, G2!',
  })

  bridge.createStartUpPageContainer(new CreateStartUpPageContainer({
    containerTotalNum: 1,
    textObject: [text],
  }))

  bridge.onEvenHubEvent((event) => {
    console.log('tap!', event)
  })
}

init()
```

Dev workflow:

```bash
npm run dev                                      # Vite on :5173
npx evenhub qr --ip 192.168.1.100 --port 5173    # QR for sideload
```

Scan QR with the Even Realities iOS/Android app → app loads on glasses with hot reload.

Read the **[full getting started guide](docs/getting-started.md)**.

---

## Core concepts

### The architecture

```
[Your web app] --HTTPS--> [Phone WebView in Even App] --BLE--> [G2 glasses]
```

- Your app is a **standard web app** (HTML/CSS/TypeScript). No special framework.
- Runs in a **WebView** inside the native Even Realities phone app.
- The glasses are a **display + input peripheral only** — no code runs there.
- The SDK injects a JavaScript bridge (`EvenAppBridge`) into the WebView.

### The display

- **576×288 pixels**, 4-bit greyscale (16 shades of green on the physical micro-LED).
- Coordinate system: top-left origin, X increases right, Y increases down.
- No color, no font selection, no font sizing, no audio output, no camera.

### Containers

You draw by creating **containers** (text, list, or image) via `createStartUpPageContainer` (called once) then `rebuildPageContainer` (for updates) or `textContainerUpgrade` (for flicker-free text updates).

- **Max 4 containers per page** in practice (docs say 12 — don't push it).
- **Exactly one container** must have `isEventCapture: 1` or no taps will fire.
- Images must be 4-bit indexed PNG, max 200×100 pixels (see [pixel art recipe](#pixel-art-pipeline)).

### Events

Tap, double-tap, scroll up/down, and lifecycle events all arrive via `bridge.onEvenHubEvent(callback)`. **This is where most bugs hide** — see [SDK quirks](#sdk-quirks-and-workarounds).

---

## SDK quirks and workarounds

These are the non-obvious problems we discovered while shipping. **Every one of them will bite you.**

### 🐛 Bug #1: `eventType: 0` normalizes to `undefined`

`CLICK_EVENT` has value `0`. JSON parsing on the way to your callback drops the `0`, so `event.textEvent.eventType` arrives as `undefined`.

**Fix**: normalize `undefined`/`null` back to `0`:

```typescript
function normalizeEventType(raw: unknown): number {
  if (raw === undefined || raw === null) return 0  // CLICK_EVENT = 0
  if (typeof raw === 'number') return raw
  if (typeof raw === 'string') return parseInt(raw, 10) || 0
  return -1
}
```

### 🐛 Bug #2: Real hardware sends `sysEvent`, simulator sends `textEvent`

This is the most insidious bug. Your app works perfectly in the simulator but **all taps are silently dropped on real glasses**.

On real G2 hardware, tap events arrive in the `sysEvent` field of the event object. The simulator routes them through `textEvent` / `listEvent`. Naive parsers only check `textEvent` → nothing ever fires.

**Fix**: always check `sysEvent` as a fallback:

```typescript
function parseEvent(event: unknown): ParsedEvent | null {
  const e = event as Record<string, unknown>

  // Try listEvent first (has selectedIndex)
  const listEvt = e.listEvent as Record<string, unknown> | undefined
  if (listEvt && typeof listEvt === 'object') {
    return { /* ... */ }
  }

  // Try textEvent (simulator)
  const textEvt = e.textEvent as Record<string, unknown> | undefined
  if (textEvt && typeof textEvt === 'object') {
    return { /* ... */ }
  }

  // CRITICAL: Try sysEvent (real hardware sends here!)
  const sysEvt = e.sysEvent as Record<string, unknown> | undefined
  if (sysEvt && typeof sysEvt === 'object') {
    const evtType = normalizeEventType(sysEvt.eventType)
    // eventType 0-3 are user interactions; 4-8 are lifecycle/IMU
    if (evtType >= 0 && evtType <= 3) {
      return { action: eventTypeToAction(evtType), /* ... */ }
    }
  }

  // Fallback: jsonData
  const json = e.jsonData as Record<string, unknown> | undefined
  if (json && typeof json === 'object') {
    return { /* ... */ }
  }

  return null
}
```

See [`docs/event-handling.md`](docs/event-handling.md) for the full template.

### 🐛 Bug #3: `ImageContainerProperty` rejects raw RGBA PNGs

`canvas.toDataURL('image/png')` produces 24-bit RGBA PNGs. The G2 host's `imageToGray4` function silently rejects them — no error, nothing displays.

**Fix**: use [`upng-js`](https://github.com/photopea/UPNG.js/) to produce 16-color indexed PNGs, with manual greyscale quantization to 16 levels (0, 17, 34, ..., 255).

```typescript
import UPNG from 'upng-js'

function generatePNG(width: number, height: number, drawFn: (ctx) => void): number[] {
  // 1. Draw with greyscale only, quantized to 16 levels
  const rgba = new Uint8Array(width * height * 4)
  // ... fill rgba with values like 0, 17, 34, ..., 255 ...

  // 2. Encode as 16-color indexed PNG
  const png = UPNG.encode([rgba.buffer], width, height, 16)
  return Array.from(new Uint8Array(png))
}
```

Credit: this approach is from [fabioglimb/even-toolkit](https://github.com/fabioglimb/even-toolkit).

### 🐛 Bug #4: Image containers have size limits that differ from docs

Docs say `ImageContainerProperty` supports up to 288×144 px. In practice on real hardware, anything over **~200×100 px is unreliable**. Stay under that.

### 🐛 Bug #5: First list item often omits `selectedIndex`

When the user taps the first item in a list container, the event sometimes arrives without `currentSelectItemIndex`. If you don't default to 0, the user can't select the top item.

**Fix**: explicit fallback:

```typescript
selectedIndex: Number(listEvt.currentSelectItemIndex ?? 0)
```

### 🐛 Bug #6: Scroll events spam without a cooldown

The hardware fires scroll events *very* rapidly. A single finger swipe can produce 5-10 events. Throttle them:

```typescript
let lastScrollTime = 0
const SCROLL_COOLDOWN_MS = 300

if (action === 'scrollUp' || action === 'scrollDown') {
  const now = Date.now()
  if (now - lastScrollTime < SCROLL_COOLDOWN_MS) return
  lastScrollTime = now
  // handle scroll
}
```

### 🐛 Bug #7: Images must be sent in sequence, not parallel

Calling `updateImageRawData()` for multiple containers concurrently locks up the bridge. Queue them:

```typescript
let imageBusy = false
async function safeUpdateImage(...) {
  if (imageBusy) return
  imageBusy = true
  try { await bridge.updateImageRawData(...) }
  finally { imageBusy = false }
}
```

### 🐛 Bug #8: Image containers must be declared before `updateImageRawData`

You cannot send image data to a container that wasn't declared in the most recent `createStartUpPageContainer` or `rebuildPageContainer`. Sequence:

```typescript
// 1. First, ensure a page exists (text-only on first send)
bridge.createStartUpPageContainer(new CreateStartUpPageContainer({ ... }))

// 2. Now rebuild with image container declared (empty placeholder)
bridge.rebuildPageContainer(new RebuildPageContainer({
  containerTotalNum: 2,
  textObject: [text],
  imageObject: [new ImageContainerProperty({ containerID: 1, containerName: 'img', ... })],
}))

// 3. SHORT DELAY (100ms) before pushing image data
await new Promise(r => setTimeout(r, 100))

// 4. Now push the PNG bytes
await bridge.updateImageRawData(new ImageRawDataUpdate({
  containerID: 1, containerName: 'img', imageData: pngBytes,
}))
```

Skipping step 1 or 3 causes silent failures.

### 🐛 Bug #9: Apps keep running in background but cannot render

When the user navigates away from your app on the glasses, JavaScript timers keep firing but any `rebuildPageContainer` calls are silently dropped. This wastes battery. **Handle `FOREGROUND_EXIT_EVENT` (4)** and pause your timers; resume on `FOREGROUND_ENTER_EVENT` (5).

### 🐛 Bug #10: There is no background / scheduled-launch API

The SDK v0.0.9 has **no notification, push, scheduled, or wakeup API** for third-party apps. Your app must be in the foreground to deliver alerts. If you need reminder-style notifications without keeping the app open, build a **companion mobile app** with `expo-notifications` — the notifications forward to the G2 automatically via the native Even app (when the user enables "Notification Access").

Full list + explanations: [`docs/sdk-quirks.md`](docs/sdk-quirks.md).

---

## Recipes

### Event handling template

[`docs/event-handling.md`](docs/event-handling.md) — drop-in `parseEvent` + handler skeleton with sysEvent/normalizer/lifecycle wired in.

### Pixel art pipeline

[`docs/pixel-art.md`](docs/pixel-art.md) — full sprite → PNG → `updateImageRawData` flow with upng-js, ASCII sprite definitions, greyscale quantization.

### Lifecycle management

[`docs/lifecycle.md`](docs/lifecycle.md) — foreground/background, device info, battery awareness, graceful shutdown, launch source differentiation.

### Distributed backend pattern

[`docs/distributed-backend.md`](docs/distributed-backend.md) — when and how to offload work to a backend server (used by Speech Coach for real Whisper STT).

### Mobile companion pattern

[`docs/mobile-companion.md`](docs/mobile-companion.md) — Expo app + `expo-notifications` that forwards to glasses via Even's native notification bridge. Reminder-style apps should use this pattern.

### Testing

[`docs/testing.md`](docs/testing.md) — state/logic test suite pattern using `tsx` (no Jest setup needed).

### Dev workflow

[`docs/dev-workflow.md`](docs/dev-workflow.md) — hot reload with `evenhub qr`, simulator vs hardware gotchas, debug patterns.

### Error telemetry

[`docs/telemetry.md`](docs/telemetry.md) — Cloudflare Worker + client for remote error reporting.

---

## Example projects

Eight production apps built following the patterns in this guide. Each is open source on GitHub.

| Project | What it does | Techniques shown |
|---|---|---|
| **[eyefit-g2](https://github.com/aleapc/eyefit-g2)** | Eye exercise guidance | Pixel art character, IMU head tracking, lifecycle events |
| **[hunter-g2](https://github.com/aleapc/hunter-g2)** | Place discovery with walking routes | Category pixel art icons, offline cache, OSRM routing, favorites |
| **[speechcoach-g2](https://github.com/aleapc/speechcoach-g2)** | Live speech pace coaching | Audio capture, backend STT via SSE, pixel art mascot, VU meter |
| **[breakmate-g2](https://github.com/aleapc/breakmate-g2)** | Health reminders | Animated pixel art character, multi-frame walking animation |
| **[breakmate-mobile](https://github.com/aleapc/breakmate-mobile)** | Expo companion for BreakMate | expo-notifications, calendar integration, smart scheduling, quiet hours |
| **[eyefit-mobile](https://github.com/aleapc/eyefit-mobile)** | Expo companion for EyeFit | expo-notifications, scheduled reminders |
| **[speechcoach-backend](https://github.com/aleapc/speechcoach-backend)** | Node backend for STT | Express + SSE, Whisper/Deepgram integration, session store |
| **[g2-telemetry-worker](https://github.com/aleapc/g2-telemetry-worker)** | Error reporting endpoint | Cloudflare Worker, KV storage, CORS |

---

## Community & credits

This guide stands on the shoulders of the open-source G2 developer community. Huge thanks to:

- **[BxNxM](https://github.com/BxNxM)** — [even-dev](https://github.com/BxNxM/even-dev) is THE reference simulator + 16 example apps. The fastest way for newcomers to get started.
- **[nickustinov](https://github.com/nickustinov)** — [even-g2-notes](https://github.com/nickustinov/even-g2-notes) is the community documentation bible. [paddle-even-g2](https://github.com/nickustinov/paddle-even-g2) shows clean minimal game architecture.
- **[fabioglimb](https://github.com/fabioglimb)** — [even-toolkit](https://github.com/fabioglimb/even-toolkit) pioneered the working pixel art recipe with upng-js. This entire guide's pixel art section is built on their work.
- **[kqb](https://github.com/kqb)** — [openclaw-g2-hud](https://github.com/kqb/openclaw-g2-hud) is a production-quality reference for audio capture + WebSocket backend + notifications.
- **[sam-siavoshian](https://github.com/sam-siavoshian)** — [claude-code-g2](https://github.com/sam-siavoshian/claude-code-g2) demonstrates distributed architecture (backend + phone WebView + glasses).
- **[200even](https://github.com/200even)** — [flappy-g2](https://github.com/200even/flappy-g2) minimal game reference.
- **[i-soxi](https://github.com/i-soxi)** — [even-g2-protocol](https://github.com/i-soxi/even-g2-protocol) reverse engineering of the BLE protocol, for when you need to go below the SDK.
- **[Even Realities](https://github.com/even-realities)** — [EvenDemoApp](https://github.com/even-realities/EvenDemoApp) and [EH-InNovel](https://github.com/even-realities/EH-InNovel) as official references.

If your project should be listed here, open a PR or issue.

---

## Contributing

PRs welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for details.

Particularly needed:
- **Counter-examples**: "Bug #X does NOT happen in SDK v0.0.Y" — we want to know when bugs get fixed
- **More recipes**: anything you struggled with and figured out
- **Translations**: this guide is English; translations to other languages are welcome
- **Corrections**: if anything here is wrong or outdated, please fix it

---

## License

MIT. Use freely, share freely.

## Disclaimer

This guide is **not affiliated with Even Realities**. It's a community project documenting what actually works based on shipping real apps. The SDK may change; quirks may be fixed in future versions. We try to keep things up to date but can't guarantee it.
