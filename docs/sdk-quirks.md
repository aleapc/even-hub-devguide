# SDK Quirks Reference

Complete list of known quirks, bugs, and workarounds in Even Hub SDK v0.0.9. Each entry includes a description, impact, reproduction, and fix.

## Event handling

### Quirk 1: `eventType: 0` normalizes to `undefined`

**Description**: `CLICK_EVENT` has value `0`. JSON serialization drops the `0`, so your callback receives `undefined` instead.

**Impact**: A naive `if (event.eventType === 0)` check never matches → click handler never fires.

**Fix**: Normalize `undefined`/`null` back to `0` at parse time. See [event-handling.md](event-handling.md).

---

### Quirk 2: Real hardware sends events via `sysEvent`, simulator via `textEvent`

**Description**: On real G2 glasses, tap/scroll events arrive in `event.sysEvent.eventType`. On the simulator, they arrive in `event.textEvent.eventType` (or `event.listEvent.eventType` for lists). **This single difference causes 90% of "works in sim, broken on hardware" bugs.**

**Impact**: Apps tested only in the simulator will fail to receive any input on real hardware. Silently. With no error.

**Reproduction**: Build a handler that only checks `event.textEvent` → works in sim, dead on hardware.

**Fix**: Always check `sysEvent` as a fallback AND accept that it may be the primary source on real hardware. See [event-handling.md](event-handling.md).

---

### Quirk 3: First list item often omits `selectedIndex`

**Description**: When a user taps the first item (index 0) in a `ListContainerProperty`, the event sometimes arrives without `currentSelectItemIndex` set.

**Impact**: If you treat missing index as "invalid", the user cannot select the top of any list.

**Fix**: Explicit default: `Number(listEvt.currentSelectItemIndex ?? 0)`.

---

### Quirk 4: Scroll events fire rapidly without throttling

**Description**: A single finger swipe on the temple touch pad can produce 5-10 `SCROLL_TOP_EVENT` / `SCROLL_BOTTOM_EVENT` dispatches.

**Impact**: Naive scroll handling advances a cursor 5+ positions per swipe.

**Fix**: 300 ms cooldown between scroll events. See [event-handling.md](event-handling.md).

---

## Image rendering

### Quirk 5: `canvas.toDataURL('image/png')` output is rejected by the host

**Description**: Standard canvas PNG export produces 24-bit RGBA. The G2 host's `imageToGray4` function silently rejects these. No error. Container shows nothing.

**Impact**: Hours of debugging while wondering why your image doesn't render.

**Fix**: Use `upng-js` with 16-color indexed PNG output. See [pixel-art.md](pixel-art.md).

---

### Quirk 6: Image max size is ~200×100, not 288×144

**Description**: SDK docs say `ImageContainerProperty` supports up to 288×144 pixels. Empirically, images over ~200×100 fail on real hardware.

**Impact**: Images work in simulator, fail on hardware.

**Fix**: Stay under 200×100 pixels. Our projects use 72×96 as a safe working size.

---

### Quirk 7: Image data cannot be sent in `createStartUpPageContainer`

**Description**: You can declare an `ImageContainerProperty` in `createStartUpPageContainer`, but you **cannot** send image data there. The subsequent `updateImageRawData` call will fail.

**Impact**: Apps that try to render images on first load end up with blank containers.

**Fix**: Use text-only in the first `createStartUpPageContainer` call, then `rebuildPageContainer` with the image container, then `updateImageRawData`. See [pixel-art.md](pixel-art.md).

---

### Quirk 8: `updateImageRawData` must not be called concurrently

**Description**: Parallel `updateImageRawData` calls for different containers lock up the bridge.

**Impact**: Sending multiple images simultaneously → no images render, bridge becomes unresponsive until the app restarts.

**Fix**: Use a busy flag or promise queue. One image at a time.

---

### Quirk 9: Short delay required between page rebuild and image update

**Description**: Calling `updateImageRawData` immediately after `rebuildPageContainer` sometimes fails because the container isn't registered yet.

**Impact**: First image update silently fails.

**Fix**: `await new Promise(r => setTimeout(r, 100))` between the two calls.

---

## Page / container management

### Quirk 10: Maximum 4 containers per page (not 12 as docs say)

**Description**: SDK docs say `containerTotalNum` can go up to 12. In practice, more than 4 containers per page degrades reliability.

**Impact**: Complex layouts with many containers render unpredictably.

**Fix**: Stick to ≤4 containers per page. Break complex layouts across screens.

---

### Quirk 11: Exactly one container must have `isEventCapture: 1`

**Description**: If no container has `isEventCapture: 1`, no taps are received. If multiple have it, behavior is undefined.

**Impact**: Pages with only image containers (image containers don't support event capture) receive no input.

**Fix**: Always include at least one text container with `isEventCapture: 1`. For image-heavy pages, add a full-screen transparent text container beneath the image that captures events.

---

### Quirk 12: `textContainerUpgrade` is essential for flicker-free updates

**Description**: `rebuildPageContainer` causes a visible flicker. For frequent updates (timers, counters, live data), use `textContainerUpgrade` which updates text in-place without rebuilding.

**Impact**: Per-second rebuilds (timer counters, countdowns) make the display unusable.

**Fix**: Use `textContainerUpgrade` for any update that doesn't change the layout. Example:

```typescript
bridge.textContainerUpgrade(new TextContainerUpgrade({
  containerID: 0,
  containerName: 'main',
  content: `Countdown: ${seconds}s`,
}))
```

---

## Background / lifecycle

### Quirk 13: No background / scheduled-launch API

**Description**: Third-party apps cannot be launched by schedules, wake-ups, or push notifications. Your app must be in the foreground to render anything.

**Impact**: Reminder-style apps (alarms, periodic alerts) fundamentally cannot work as stand-alone G2 apps.

**Fix**: Build a companion mobile app using `expo-notifications`. Notifications scheduled on the phone are forwarded to the glasses by the native Even Realities app (when the user enables "Notification Access"). See [mobile-companion.md](mobile-companion.md).

---

### Quirk 14: Timers keep firing in background but render calls are dropped

**Description**: When the user navigates away from your app, the WebView's JavaScript continues running. `setInterval`/`setTimeout` still fire, but any `rebuildPageContainer` / `updateImageRawData` calls are silently dropped (they're consumed by the system but never reach the display).

**Impact**: Battery is wasted running computations that can't be displayed.

**Fix**: Handle `FOREGROUND_EXIT_EVENT` (4) to pause timers and `FOREGROUND_ENTER_EVENT` (5) to resume them.

---

### Quirk 15: No graceful exit without `shutDownPageContainer`

**Description**: There's no hardware "back" or "exit" gesture that terminates a G2 app. The user has to open the glasses menu and switch away.

**Impact**: Users can't cleanly exit your app from within it.

**Fix**: Provide a gesture (triple-tap, long-press, double-swipe) that calls `bridge.shutDownPageContainer(0)`. See [lifecycle.md](lifecycle.md).

---

### Quirk 16: `onLaunchSource` fires once per page load

**Description**: `bridge.onLaunchSource((source) => {...})` tells you whether the app was launched from the phone app menu (`'appMenu'`) or the glasses menu (`'glassesMenu'`). The callback fires **once**, not again on reload.

**Impact**: You can use this to differentiate UX — e.g. skip the welcome screen if the user launched directly from the glasses (they know what the app does).

**Fix**: Subscribe early in init, don't expect re-triggers.

---

## Audio / hardware

### Quirk 17: `audioControl` requires `createStartUpPageContainer` first

**Description**: Calling `bridge.audioControl(true)` before the startup page exists fails silently.

**Impact**: Microphone never turns on.

**Fix**: Always create the page first, then enable audio.

---

### Quirk 18: G2 has NO speaker hardware

**Description**: Despite having a microphone, the G2 glasses have **no speaker**. `audioControl` only gates the mic; there's no API for audio playback.

**Impact**: Apps cannot play audio chimes, feedback tones, or voice prompts on the glasses themselves.

**Fix**: For audio feedback, use the companion phone app (via notifications) or rely on haptic cues. For visual feedback on the glasses, use screen flashes or animations.

---

### Quirk 19: IMU data arrives as `sysEvent` with `eventType: 8`

**Description**: When `imuControl(true, pace)` is enabled, IMU data (x/y/z acceleration) streams via `sysEvent` events with `eventType === 8` (`IMU_DATA_REPORT`).

**Impact**: Your event handler needs to specifically route eventType 8 to an IMU handler instead of treating it as an unknown event.

**Fix**: Check `eventType === 8` and extract `sysEvt.imuData.{x,y,z}`. See [event-handling.md](event-handling.md).

---

## Build / tooling

### Quirk 20: Supported languages list only accepts `["en", "es"]`

**Description**: The `supported_languages` field in `app.json` only accepts English and Spanish. **Portuguese is rejected by the Hub validator** even though the SDK itself has no language restrictions.

**Impact**: Apps with Portuguese UI cannot declare Portuguese in the manifest.

**Fix**: Declare `["en", "es"]` in the manifest but handle Portuguese internally via `navigator.language` detection.

---

### Quirk 21: `new EvenAppBridge()` constructor is private

**Description**: The constructor of `EvenAppBridge` is not exposed. Attempting `new EvenAppBridge()` produces a TypeScript error or runtime error.

**Impact**: Direct instantiation fails.

**Fix**: Always use `const bridge = await waitForEvenAppBridge()`.

---

## Simulator vs hardware

### Quirk 22: Simulator display is not 1:1 with hardware

**Description**: The simulator uses a system font that's different from the G2 firmware font. Text sizing, spacing, and alignment differ slightly. Layouts that look perfect in the simulator may wrap differently on hardware.

**Impact**: Visual bugs that only appear on hardware.

**Fix**: Test on real hardware before any release. Budget for minor text wrapping fixes.

---

### Quirk 23: Simulator image processing is faster and more permissive

**Description**: The simulator doesn't enforce the same image size limits as hardware, and processes images much faster.

**Impact**: Large images work in the simulator but fail on hardware.

**Fix**: Stick to the hardware limits (200×100 images) even when the simulator accepts more.

---

### Quirk 24: Simulator never emits IMU or audio data

**Description**: `imuControl(true)` and `audioControl(true)` succeed in the simulator, but no actual data is ever delivered.

**Impact**: IMU/audio features cannot be tested in the simulator.

**Fix**: Mock IMU/audio data for unit tests. Full testing requires real hardware.

---

## Contributing

Found a new quirk? Please open a PR or issue. Include:

1. Description (what behavior do you observe?)
2. SDK version (`@evenrealities/even_hub_sdk` version)
3. Reproduction steps or minimal code sample
4. Fix or workaround (if known)

We want this to be the most honest, up-to-date reference for actually shipping G2 apps.
