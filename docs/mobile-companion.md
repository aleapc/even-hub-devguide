# Mobile Companion Pattern

For apps that need **scheduled alerts** (reminders, notifications, time-based triggers), the Even Hub SDK v0.0.9 has **no background/push API**. Your G2 app must be in the foreground to deliver any visual output.

The workaround: build a **companion mobile app** that schedules local notifications on the phone. The native Even Realities app forwards those notifications to the glasses automatically, as long as the user has "Notification Access for Even App" enabled in their phone settings.

This is how **BreakMate Mobile** and **EyeFit Mobile** work.

## Architecture

```
[Your mobile app (Expo)]
    ↓ schedules local notifications
[Phone OS notification system]
    ↓ fires at scheduled time
[Native Even Realities app]
    ↓ forwards via BLE
[G2 glasses display]
```

Crucially, **your mobile app does not need to communicate directly with the glasses**. The native Even Realities app handles that part. You just schedule standard local notifications.

## Stack

- **Expo SDK 54+** with TypeScript
- **expo-notifications** for scheduled local notifications
- **@react-native-async-storage/async-storage** for state persistence
- (Optional) **expo-calendar** for smart scheduling

## Scaffold a new companion

```bash
cd /d/dev
npx create-expo-app@latest my-app-mobile --template blank-typescript --no-install
cd my-app-mobile
npm install
npx expo install expo-notifications expo-device expo-constants @react-native-async-storage/async-storage
```

## Minimum viable companion

`src/notifications.ts`:

```typescript
import * as Notifications from 'expo-notifications'
import * as Device from 'expo-device'
import { Platform } from 'react-native'

// Configure how notifications appear when app is in foreground
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowBanner: true,
    shouldShowList: true,
    shouldPlaySound: true,
    shouldSetBadge: false,
  }),
})

export async function requestPermissions(): Promise<boolean> {
  if (!Device.isDevice) {
    console.warn('Notifications only work on physical devices')
    return false
  }

  const { status: existing } = await Notifications.getPermissionsAsync()
  let status = existing
  if (status !== 'granted') {
    const result = await Notifications.requestPermissionsAsync()
    status = result.status
  }
  if (status !== 'granted') return false

  // Android: create a notification channel
  if (Platform.OS === 'android') {
    await Notifications.setNotificationChannelAsync('reminders', {
      name: 'Reminders',
      importance: Notifications.AndroidImportance.HIGH,
      vibrationPattern: [0, 250, 250, 250],
      lightColor: '#2d8c4e',
      sound: 'default',
    })
  }

  return true
}

export async function scheduleReminder(
  title: string,
  body: string,
  secondsFromNow: number,
  repeats = false,
): Promise<string> {
  return await Notifications.scheduleNotificationAsync({
    content: {
      title,
      body,
      sound: 'default',
    },
    trigger: {
      type: Notifications.SchedulableTriggerInputTypes.TIME_INTERVAL,
      seconds: secondsFromNow,
      repeats,
      channelId: 'reminders',
    },
  })
}

export async function cancelAll(): Promise<void> {
  await Notifications.cancelAllScheduledNotificationsAsync()
}
```

## Notification actions (hands-free interaction)

For richer UX, register notification categories with action buttons. Users can tap **Done** or **Snooze** directly from the notification without opening your app.

```typescript
export async function registerCategories(): Promise<void> {
  await Notifications.setNotificationCategoryAsync('my-reminder', [
    {
      identifier: 'done',
      buttonTitle: 'Done ✓',
      options: { opensAppToForeground: false },
    },
    {
      identifier: 'snooze',
      buttonTitle: 'Snooze 5m',
      options: { opensAppToForeground: false },
    },
  ])
}
```

Schedule with the category:

```typescript
await Notifications.scheduleNotificationAsync({
  content: {
    title: 'Time to rest your eyes',
    body: 'Look at something 20ft away for 20 seconds',
    categoryIdentifier: 'my-reminder',
    sound: 'default',
  },
  trigger: { /* ... */ },
})
```

Handle the action:

```typescript
import { useEffect } from 'react'
import * as Notifications from 'expo-notifications'

// Inside App.tsx
useEffect(() => {
  const sub = Notifications.addNotificationResponseReceivedListener((response) => {
    const action = response.actionIdentifier
    if (action === 'done') {
      // Increment stats, play success animation, etc.
    } else if (action === 'snooze') {
      // Schedule a new one-time notification 5 minutes later
    }
  })
  return () => sub.remove()
}, [])
```

## Smart scheduling with calendar integration

For reminders that should respect the user's meetings:

```bash
npx expo install expo-calendar
```

```typescript
import * as Calendar from 'expo-calendar'

export async function isInMeeting(): Promise<boolean> {
  const { status } = await Calendar.getCalendarPermissionsAsync()
  if (status !== 'granted') return false

  const now = new Date()
  const in5min = new Date(now.getTime() + 5 * 60 * 1000)
  const calendars = await Calendar.getCalendarsAsync(Calendar.EntityTypes.EVENT)
  const events = await Calendar.getEventsAsync(
    calendars.map(c => c.id),
    now,
    in5min,
  )
  return events.some(e => !e.allDay && new Date(e.startDate) <= now)
}
```

Use it in the notification listener:

```typescript
Notifications.addNotificationReceivedListener(async (notification) => {
  if (!respectCalendar) return
  if (await isInMeeting()) {
    // Dismiss and reschedule 5 min later
    await Notifications.dismissNotificationAsync(notification.request.identifier)
    // Schedule a one-time snooze
  }
})
```

**Limitation**: this listener only runs while the app is in memory (foreground or recent background). When the OS fully terminates the app, notifications fire directly and bypass this check. For full coverage you'd need a native background task.

## User setup flow

To receive notifications on their glasses, the user needs to:

1. Install your mobile app
2. Grant notification permissions (prompted on first launch)
3. Install the **Even Realities** phone app
4. Open Even Realities → **Settings → Notification Access**
5. Enable notification forwarding for your mobile app

Document this clearly in your app's onboarding.

## iOS vs Android considerations

**Android**:
- Notification channels give you fine-grained control
- Action buttons work reliably
- `expo-calendar` integration is straightforward
- Background tasks can extend notification listener coverage

**iOS**:
- Focus modes can suppress your notifications — warn users
- Background task support is more limited
- Notification actions only show in expanded view (swipe)
- Quiet hours feature should align with iOS Do Not Disturb

Both platforms support `expo-notifications` equally well for basic scheduled reminders.

## Examples in this ecosystem

- **[breakmate-mobile](https://github.com/aleapc/breakmate-mobile)** — 5 reminder types, calendar integration, quiet hours, action buttons
- **[eyefit-mobile](https://github.com/aleapc/eyefit-mobile)** — Exercise session reminders with weekday-only schedule

Both are minimal enough to read cover-to-cover in 30 minutes.

## When NOT to use this pattern

- If your app needs to display rich interactive content (not just a notification)
- If you need truly live data (the mobile notification is a one-way push)
- If you want to avoid the "Notification Access" friction in user onboarding

For those cases, you'll need a G2 app that stays in the foreground and uses a real-time data source (like a backend WebSocket). See [distributed-backend.md](distributed-backend.md).
