# Event Handling Template

This is a drop-in, production-tested event handler that correctly handles all the SDK quirks discussed in the [main README](../README.md#sdk-quirks-and-workarounds).

Copy `src/events.ts` into your project as-is and adapt the screen logic at the bottom to your app.

## Full template

```typescript
// src/glasses/events.ts — production-tested event handler
// Handles: sysEvent (real hardware), eventType 0 → undefined (SDK quirk),
// lifecycle events, scroll cooldown, triple-tap graceful shutdown

import type { EvenAppBridge } from '@evenrealities/even_hub_sdk'
import type { AppState } from '../state'

// Event type constants (from OsEventTypeList)
const EVT_CLICK = 0
const EVT_SCROLL_TOP = 1
const EVT_SCROLL_BOTTOM = 2
const EVT_DOUBLE_CLICK = 3
const EVT_FOREGROUND_ENTER = 4
const EVT_FOREGROUND_EXIT = 5
const EVT_ABNORMAL_EXIT = 6
const EVT_SYSTEM_EXIT = 7
const EVT_IMU_DATA_REPORT = 8

// Throttling
const SCROLL_COOLDOWN_MS = 300
const TRIPLE_TAP_WINDOW_MS = 600

interface ParsedEvent {
  action: 'click' | 'doubleClick' | 'scrollUp' | 'scrollDown' | 'foregroundEnter' | 'foregroundExit' | 'unknown'
  containerName: string
  containerID: number
  selectedIndex: number
  imuData?: { x: number; y: number; z: number }
}

/**
 * Normalize eventType. THE MOST IMPORTANT HELPER.
 * SDK quirk: eventType 0 (CLICK_EVENT) serializes as undefined.
 */
function normalizeEventType(raw: unknown): number {
  if (raw === undefined || raw === null) return EVT_CLICK
  if (typeof raw === 'number') return raw
  if (typeof raw === 'string') return parseInt(raw, 10) || 0
  return -1
}

function eventTypeToAction(evtType: number): ParsedEvent['action'] {
  switch (evtType) {
    case EVT_CLICK: return 'click'
    case EVT_SCROLL_TOP: return 'scrollUp'
    case EVT_SCROLL_BOTTOM: return 'scrollDown'
    case EVT_DOUBLE_CLICK: return 'doubleClick'
    case EVT_FOREGROUND_ENTER: return 'foregroundEnter'
    case EVT_FOREGROUND_EXIT: return 'foregroundExit'
    default: return 'unknown'
  }
}

/**
 * Robust event parser. Tries all known event shapes:
 *   listEvent → textEvent → sysEvent (REAL HARDWARE) → jsonData
 *
 * CRITICAL: sysEvent must be checked even if listEvent/textEvent don't match,
 * because REAL G2 GLASSES send tap events via sysEvent, not textEvent.
 * The simulator sends via textEvent, masking this bug during development.
 */
export function parseEvent(event: unknown): ParsedEvent | null {
  const e = event as Record<string, unknown>

  // Try listEvent first (has selectedIndex for list taps)
  const listEvt = e.listEvent as Record<string, unknown> | undefined
  if (listEvt && typeof listEvt === 'object') {
    return {
      action: eventTypeToAction(normalizeEventType(listEvt.eventType)),
      containerName: String(listEvt.containerName ?? ''),
      containerID: Number(listEvt.containerID ?? 0),
      selectedIndex: Number(listEvt.currentSelectItemIndex ?? 0),
    }
  }

  // Try textEvent (simulator)
  const textEvt = e.textEvent as Record<string, unknown> | undefined
  if (textEvt && typeof textEvt === 'object') {
    return {
      action: eventTypeToAction(normalizeEventType(textEvt.eventType)),
      containerName: String(textEvt.containerName ?? ''),
      containerID: Number(textEvt.containerID ?? 0),
      selectedIndex: 0,
    }
  }

  // CRITICAL: Try sysEvent (REAL HARDWARE sends here!)
  const sysEvt = e.sysEvent as Record<string, unknown> | undefined
  if (sysEvt && typeof sysEvt === 'object') {
    const evtType = normalizeEventType(sysEvt.eventType)

    // IMU data comes via sysEvent with eventType 8
    if (evtType === EVT_IMU_DATA_REPORT) {
      const imu = sysEvt.imuData as { x?: number; y?: number; z?: number } | undefined
      return {
        action: 'unknown',
        containerName: '',
        containerID: 0,
        selectedIndex: 0,
        imuData: imu ? { x: imu.x ?? 0, y: imu.y ?? 0, z: imu.z ?? 0 } : undefined,
      }
    }

    // User interaction events (0-5)
    if (evtType >= 0 && evtType <= 5) {
      return {
        action: eventTypeToAction(evtType),
        containerName: String(sysEvt.containerName ?? ''),
        containerID: Number(sysEvt.containerID ?? 0),
        selectedIndex: Number(sysEvt.currentSelectItemIndex ?? 0),
      }
    }
  }

  // Fallback: jsonData
  const json = e.jsonData as Record<string, unknown> | undefined
  if (json && typeof json === 'object') {
    return {
      action: eventTypeToAction(normalizeEventType(json.eventType)),
      containerName: String(json.containerName ?? ''),
      containerID: Number(json.containerID ?? 0),
      selectedIndex: Number(json.currentSelectItemIndex ?? 0),
    }
  }

  return null
}

// ---- Main setup ----

export interface EventHandlerCallbacks {
  onForegroundEnter?: () => void
  onForegroundExit?: () => void
  onIMUData?: (x: number, y: number, z: number) => void
  onTripleTap?: () => void  // for graceful shutdown gesture
}

export function setupEventHandler(
  bridge: EvenAppBridge,
  state: AppState,
  callbacks: EventHandlerCallbacks = {},
): void {
  let lastScrollTime = 0
  let tapTimestamps: number[] = []

  bridge.onEvenHubEvent((event: unknown) => {
    const parsed = parseEvent(event)
    if (!parsed) return

    // IMU data path
    if (parsed.imuData && callbacks.onIMUData) {
      callbacks.onIMUData(parsed.imuData.x, parsed.imuData.y, parsed.imuData.z)
      return
    }

    // Lifecycle events
    if (parsed.action === 'foregroundEnter') {
      callbacks.onForegroundEnter?.()
      return
    }
    if (parsed.action === 'foregroundExit') {
      callbacks.onForegroundExit?.()
      return
    }

    // Scroll events: throttled
    if (parsed.action === 'scrollUp' || parsed.action === 'scrollDown') {
      const now = Date.now()
      if (now - lastScrollTime < SCROLL_COOLDOWN_MS) return
      lastScrollTime = now
      handleScroll(bridge, state, parsed.action === 'scrollUp' ? -1 : 1)
      return
    }

    // Double-tap
    if (parsed.action === 'doubleClick') {
      handleDoubleTap(bridge, state)
      return
    }

    // Single tap with triple-tap detection
    if (parsed.action === 'click') {
      const now = Date.now()
      tapTimestamps = tapTimestamps.filter((t) => now - t < TRIPLE_TAP_WINDOW_MS)
      tapTimestamps.push(now)

      if (tapTimestamps.length >= 3) {
        tapTimestamps = []
        callbacks.onTripleTap?.()
        return
      }

      handleTap(bridge, state, parsed)
    }
  })
}

// ---- App-specific handlers (customize these) ----

function handleTap(bridge: EvenAppBridge, state: AppState, event: ParsedEvent): void {
  // Your app's tap logic here
}

function handleDoubleTap(bridge: EvenAppBridge, state: AppState): void {
  // Your app's double-tap logic (usually "go home" or "back")
}

function handleScroll(bridge: EvenAppBridge, state: AppState, direction: number): void {
  // Your app's scroll logic (+1 down, -1 up)
}
```

## Usage in `main.ts`

```typescript
setupEventHandler(bridge, state, {
  onForegroundExit: () => pauseTimers(),
  onForegroundEnter: () => resumeTimers(bridge, state),
  onTripleTap: () => {
    stopAllTimers()
    bridge.shutDownPageContainer(0)
  },
  onIMUData: (x, y, z) => {
    // Optional: feed into an IMU tracker for head movement detection
  },
})
```

## Why this works

1. **`normalizeEventType` handles the eventType-0-becomes-undefined bug** at a single chokepoint — any path (text, sys, list, json) benefits.
2. **`sysEvent` is checked** so real hardware taps fire (the simulator never exposed this).
3. **Scroll cooldown** prevents the rapid-fire scroll events from overwhelming your screen logic.
4. **Triple-tap detection** gives users a clean gesture to exit the app (`shutDownPageContainer`), since the SDK has no back button on the glasses themselves.
5. **Lifecycle events** route to your pause/resume callbacks so battery isn't wasted in the background.
6. **IMU data** routes to a separate callback so head-movement exercises work without cluttering the main input path.
