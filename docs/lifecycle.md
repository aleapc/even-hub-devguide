# Lifecycle Management

Your G2 app needs to handle four things correctly or it wastes battery, loses state, and frustrates users:

1. **Foreground / background events** — pause timers when the user isn't looking
2. **Device status** — react to battery, wearing state
3. **Graceful shutdown** — let the user exit cleanly
4. **Launch source** — differentiate phone-menu vs glasses-menu launch

## Foreground / background events

When the user navigates away from your app (switches to another G2 app, removes the glasses, etc.), `FOREGROUND_EXIT_EVENT` fires. Your JavaScript timers **keep running** in the WebView but any visual updates are silently dropped by the system.

```typescript
// See event-handling.md for the full parser
setupEventHandler(bridge, state, {
  onForegroundExit: () => {
    pauseTimers()
    saveState(bridge, state)
  },
  onForegroundEnter: () => {
    resumeTimers(bridge, state)
  },
})
```

### Implementing pause/resume

```typescript
// In your reminders/timers module
let timerRemaining: Record<string, number> = {}
let timerStartedAt: Record<string, number> = {}

export function pauseTimers(): void {
  const now = Date.now()
  for (const [name, started] of Object.entries(timerStartedAt)) {
    const originalDuration = timerDurations[name]
    const elapsed = now - started
    const remaining = Math.max(0, originalDuration - elapsed)
    timerRemaining[name] = remaining
    clearTimeout(timerHandles[name])
  }
  timerStartedAt = {}
}

export function resumeTimers(bridge, state): void {
  const now = Date.now()
  for (const [name, remaining] of Object.entries(timerRemaining)) {
    timerHandles[name] = setTimeout(() => { /* fire timer */ }, remaining)
    timerStartedAt[name] = now
  }
  timerRemaining = {}
}
```

## Device info & status changes

The SDK exposes `getDeviceInfo()` and `onDeviceStatusChanged()` but almost no app uses them. Shame, because they enable:

- **Battery level display**: show a small `[85%]` in a corner
- **Low-battery mode**: reduce animations below 10%
- **Wearing detection**: pause timers when glasses are removed (`isWearing === false`)
- **Connection monitoring**: warn when BLE drops

```typescript
try {
  const device = await bridge.getDeviceInfo()
  const status = device?.status
  state.batteryLevel = status?.batteryLevel ?? 100
  state.isWearing = status?.isWearing ?? true
} catch {
  // Simulator or old SDK — use defaults
}

bridge.onDeviceStatusChanged((status) => {
  state.batteryLevel = status.batteryLevel ?? state.batteryLevel
  state.isWearing = status.isWearing ?? state.isWearing

  // React: low battery → skip animations
  if (state.batteryLevel < 10) {
    setLowBatteryMode(true)
  }

  // React: glasses removed → pause
  if (state.isWearing === false) {
    pauseTimers()
  } else {
    resumeTimers(bridge, state)
  }
})
```

### Battery badge on screen

Show a small `[NN%]` in the corner of your home screen:

```typescript
const battery = state.batteryLevel < 15 ? `[!${state.batteryLevel}%]` : `[${state.batteryLevel}%]`
const content = `MyApp                         ${battery}\n\n...rest of content`
```

## Graceful shutdown

There's no hardware back button on the G2. The user has to navigate to the glasses menu and switch apps. You should provide a gesture to exit cleanly:

```typescript
// In setupEventHandler callbacks:
onTripleTap: () => {
  clearAnimation()
  stopAllTimers()
  try {
    bridge.shutDownPageContainer(0)  // 0 = immediate exit
  } catch (e) {
    console.error('shutdown failed:', e)
  }
},
```

Alternatives:
- Long-press (hold tap for 1s)
- Double-swipe-down
- Tap-and-hold a "X" icon in the corner

Pick whatever is natural for your app. Document it in your home screen footer: `Triple-tap = exit`.

## Launch source

`bridge.onLaunchSource((source) => {...})` tells you how the app was launched:

- `'appMenu'` — user opened the Even Realities phone app and tapped your app in their plugin list
- `'glassesMenu'` — user navigated to your app directly from the glasses menu

This fires **once** per page load. Use it to differentiate UX:

```typescript
let launchedFromGlasses = false

bridge.onLaunchSource((source) => {
  launchedFromGlasses = source === 'glassesMenu'
})

// Later, when first rendering:
if (launchedFromGlasses) {
  // User is already at the glasses — skip the welcome screen, jump to main function
  showMainScreen(bridge, state)
} else {
  // User tapped from phone — show welcome or onboarding
  showWelcomeScreen(bridge, state)
}
```

## Complete init flow

Here's what a properly lifecycle-aware `main.ts` looks like:

```typescript
import { waitForEvenAppBridge } from '@evenrealities/even_hub_sdk'
import { setupEventHandler } from './glasses/events'
import { loadState, saveState, pauseTimers, resumeTimers, startAllTimers } from './state'

async function init() {
  const bridge = await waitForEvenAppBridge()
  const state = await loadState(bridge)

  // 1. Device info snapshot + live updates
  try {
    const device = await bridge.getDeviceInfo()
    if (device?.status) {
      state.batteryLevel = device.status.batteryLevel ?? 100
      state.isWearing = device.status.isWearing ?? true
    }
    bridge.onDeviceStatusChanged((status) => {
      state.batteryLevel = status.batteryLevel ?? state.batteryLevel
      state.isWearing = status.isWearing ?? state.isWearing
      if (state.isWearing === false) pauseTimers()
      else resumeTimers(bridge, state)
    })
  } catch { /* simulator */ }

  // 2. Launch source
  let launchedFromGlasses = false
  try {
    bridge.onLaunchSource((source) => {
      launchedFromGlasses = source === 'glassesMenu'
    })
  } catch { /* optional */ }

  // 3. Event handler with lifecycle + shutdown callbacks
  setupEventHandler(bridge, state, {
    onForegroundExit: () => {
      pauseTimers()
      saveState(bridge, state)
    },
    onForegroundEnter: () => {
      resumeTimers(bridge, state)
    },
    onTripleTap: () => {
      stopAllTimers()
      bridge.shutDownPageContainer(0)
    },
  })

  // 4. Start timers
  startAllTimers(bridge, state)

  // 5. Render initial screen based on launch source
  if (launchedFromGlasses) {
    showMainScreen(bridge, state)
  } else {
    showWelcome(bridge, state)
  }
}

init().catch(console.error)
```

## Testing lifecycle events

The simulator **does not emit lifecycle events**. You can't test this path in development. Either:

1. Test on real hardware
2. Manually dispatch fake events via the simulator's event injection API (if any)
3. Unit test the pause/resume logic directly without going through the bridge

We use approach #3 for unit tests — see [testing.md](testing.md) — and hand-verify on real hardware for integration.
