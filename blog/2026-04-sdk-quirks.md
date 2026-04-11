# 5 Even Hub SDK Quirks That Will Ruin Your Day (And How to Fix Them)

*April 2026 — notes from shipping 4 production apps on the Even Realities Even Hub platform*

I recently finished shipping **four apps** for the Even Realities G2 smart glasses: a place-discovery app, an eye-exercise coach, a speech-pace analyzer, and a health-reminder system. Along the way I hit a surprising number of undocumented SDK bugs — things that work perfectly in the simulator but silently fail on real hardware, or fail in ways that produce no error messages, or behave opposite to what the docs say.

This post is five of the worst ones, why they happen, and how to fix them. If you're building anything on the Even Hub SDK v0.0.9, read this before you start writing event handlers.

For the full reference (24 quirks and counting), see [even-hub-devguide/docs/sdk-quirks.md](https://github.com/aleapc/even-hub-devguide/blob/master/docs/sdk-quirks.md).

---

## 1. The one that will cost you two days of debugging: `sysEvent` vs `textEvent`

**What you write:**

```typescript
bridge.onEvenHubEvent((event) => {
  const textEvt = event.textEvent
  if (textEvt?.eventType === 0) {
    handleTap()
  }
})
```

**What happens:**

- Works perfectly in the simulator ✅
- Silently does nothing on real hardware ❌
- No error. No warning. No log. Just dead input.

**Why:** On real G2 glasses, tap events arrive in `event.sysEvent`, NOT `event.textEvent`. The simulator sends them via `textEvent`, which masks the bug during development. You ship to the Hub, put the `.ehpk` on your glasses, and none of your buttons work.

I lost a day to this before I built a debug build that dumped the raw event to the display. When I saw the output, the "aha" was immediate:

```
Raw: {"jsonData":{...},"sysEvent":{...}}
```

No `textEvent` anywhere.

**The fix:** always check `sysEvent` as a fallback in your event parser:

```typescript
function parseEvent(event: unknown) {
  const e = event as Record<string, unknown>

  // Try listEvent first
  const listEvt = e.listEvent as Record<string, unknown> | undefined
  if (listEvt) return parse(listEvt)

  // Try textEvent (simulator)
  const textEvt = e.textEvent as Record<string, unknown> | undefined
  if (textEvt) return parse(textEvt)

  // CRITICAL: real hardware sends here
  const sysEvt = e.sysEvent as Record<string, unknown> | undefined
  if (sysEvt) {
    const evtType = normalizeEventType(sysEvt.eventType)
    if (evtType >= 0 && evtType <= 3) return parse(sysEvt)
  }

  // Fallback: jsonData
  const json = e.jsonData as Record<string, unknown> | undefined
  if (json) return parse(json)

  return null
}
```

**Always. Test. On. Real. Hardware.**

---

## 2. `eventType: 0` normalizes to `undefined`

Closely related. `CLICK_EVENT` has enum value `0`. Something in the JSON serialization pipeline drops the `0`, so your callback sees:

```javascript
{ sysEvent: { containerName: "main", containerID: 0 } }  // note: no eventType!
```

A naive check for `eventType === 0` fails because `undefined !== 0`.

**The fix:** a single normalizer at the parse boundary:

```typescript
function normalizeEventType(raw: unknown): number {
  if (raw === undefined || raw === null) return 0  // CLICK_EVENT = 0
  if (typeof raw === 'number') return raw
  if (typeof raw === 'string') return parseInt(raw, 10) || 0
  return -1
}
```

Once you apply this at the parse boundary, you never think about it again.

---

## 3. `canvas.toDataURL('image/png')` images don't render on real hardware

**What you write:**

```typescript
const canvas = document.createElement('canvas')
canvas.width = 96
canvas.height = 128
const ctx = canvas.getContext('2d')!
// ... draw your pixel art ...
const dataUrl = canvas.toDataURL('image/png')
const bytes = base64ToBytes(dataUrl.split(',')[1])

// ImageContainer is declared on the page
await bridge.updateImageRawData({
  containerID: 1,
  containerName: 'char',
  imageData: bytes,
})
```

**What happens:** the container stays blank. No error. The host's `imageToGray4` function silently rejects the image because it's 24-bit RGBA when it expects 4-bit indexed.

**The fix:** use [`upng-js`](https://github.com/photopea/UPNG.js/) to produce **16-color indexed PNGs** with manual quantization to 16 greyscale levels (0, 17, 34, ..., 255):

```typescript
import UPNG from 'upng-js'

function quantize(grey: number): number {
  return Math.min(15, Math.round(grey / 17)) * 17
}

function generatePNG(w: number, h: number, pixels: number[]): number[] {
  const rgba = new Uint8Array(w * h * 4)
  for (let i = 0; i < pixels.length; i++) {
    const v = quantize(pixels[i])
    rgba[i * 4] = v
    rgba[i * 4 + 1] = v
    rgba[i * 4 + 2] = v
    rgba[i * 4 + 3] = 255
  }
  const png = UPNG.encode([rgba.buffer], w, h, 16)
  return Array.from(new Uint8Array(png))
}
```

Credit: this technique was pioneered by [fabioglimb/even-toolkit](https://github.com/fabioglimb/even-toolkit).

Also: **max image size is ~200×100 px**, not the 288×144 the docs advertise. Bigger images render unreliably on real hardware.

---

## 4. You cannot send image data in `createStartUpPageContainer`

**What you try:**

```typescript
bridge.createStartUpPageContainer(new CreateStartUpPageContainer({
  containerTotalNum: 2,
  textObject: [text],
  imageObject: [imageContainer],
}))
await bridge.updateImageRawData({ containerID: 1, imageData: pngBytes })
```

**What happens:** blank image container. No error.

**The fix:** the first `createStartUpPageContainer` call must be **text-only**. Then rebuild with the image container declared, wait 100ms, THEN push image data:

```typescript
// Step 1: text-only startup
bridge.createStartUpPageContainer(new CreateStartUpPageContainer({
  containerTotalNum: 1,
  textObject: [dummyText],
}))

await new Promise(r => setTimeout(r, 100))

// Step 2: rebuild with image container declared
bridge.rebuildPageContainer(new RebuildPageContainer({
  containerTotalNum: 2,
  textObject: [mainText],
  imageObject: [imageContainer],
}))

await new Promise(r => setTimeout(r, 100))

// Step 3: NOW push the image bytes
await bridge.updateImageRawData(new ImageRawDataUpdate({
  containerID: 1,
  containerName: 'char',
  imageData: pngBytes,
}))
```

Skip any step → silent failure.

---

## 5. There is no background / scheduled-launch API

This one isn't a bug — it's a fundamental architectural limit that reshapes what apps you can build.

**What's missing:** the SDK v0.0.9 has no notification API, no push API, no scheduled wakeup. Your app is either in the foreground or it's asleep. When the user navigates away:
- Your `setInterval` timers keep firing in the WebView
- But **any** `rebuildPageContainer` or `updateImageRawData` call is silently dropped
- Battery is wasted on invisible work

This means **reminder apps cannot work as stand-alone G2 apps**. BreakMate (health reminders every 20-50 min) absolutely needed a companion mobile app using `expo-notifications` — the local notifications on the phone forward to the glasses automatically via the native Even Realities app's notification-access feature.

It took me several failed attempts to accept this limitation. The workflow that actually works:

```
[Expo mobile app] → schedules local notification
       ↓
[Phone OS] → fires at scheduled time
       ↓
[Native Even Realities app] → forwards via BLE
       ↓
[G2 glasses] → shows as overlay notification
```

If your app idea requires "alert the user at time X without them opening anything," you need a mobile companion. Period. See [docs/mobile-companion.md](https://github.com/aleapc/even-hub-devguide/blob/master/docs/mobile-companion.md) for the full pattern.

---

## The meta-lesson

Writing for the Even Hub today is a bit like writing for iOS in 2008 — the platform is promising, the hardware is lovely, but the SDK is young and you'll hit undocumented sharp edges. The community has already done most of the hard reverse-engineering work: [even-dev](https://github.com/BxNxM/even-dev), [even-g2-notes](https://github.com/nickustinov/even-g2-notes), [even-toolkit](https://github.com/fabioglimb/even-toolkit), and [even-g2-protocol](https://github.com/i-soxi/even-g2-protocol) were the main sources I drew from.

**Always test on real hardware before shipping. Always.** The simulator is a decent layout iteration tool but it will lie to you about events, image rendering, lifecycle, audio, and IMU.

**Full reference** (all 24 documented quirks, working templates, recipes for pixel art, mobile companion, distributed backend, testing, telemetry): [github.com/aleapc/even-hub-devguide](https://github.com/aleapc/even-hub-devguide)

Also up on GitHub — all four apps open source, MIT licensed:
- [eyefit-g2](https://github.com/aleapc/eyefit-g2) — eye exercise coach with IMU head tracking
- [hunter-g2](https://github.com/aleapc/hunter-g2) — place discovery with pixel art category icons and OSRM routing
- [speechcoach-g2](https://github.com/aleapc/speechcoach-g2) — live speech pace coach with real Whisper STT via backend
- [breakmate-g2](https://github.com/aleapc/breakmate-g2) — animated pixel art health reminder character

If you hit a quirk not in the guide, please open a PR — this is a community doc, not a personal project.
