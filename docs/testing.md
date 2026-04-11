# Testing G2 Apps

The G2 app ecosystem doesn't have Jest/Vitest setup baked in. But you can still get serious test coverage for your **state and logic** using a lightweight pattern: a single `test-events.ts` file runnable via `tsx`.

We use this pattern across all our apps. Combined coverage: **309 passing tests** across EyeFit (107), Hunter (93), and Speech Coach (109).

## The pattern

Create `src/test-events.ts`:

```typescript
// Vanilla assertion test runner for G2 app state/logic
// Run with: npx tsx src/test-events.ts

import {
  createDefaultState,
  updateStreak,
  hadActivityToday,
  formatCountdown,
  todayStr,
  ALL_TYPES,
} from './state'

let passed = 0
let failed = 0
const failures: string[] = []

function assert(cond: boolean, msg: string): void {
  if (cond) {
    passed++
  } else {
    failed++
    failures.push(msg)
    console.error(`  ❌ ${msg}`)
  }
}

function assertEq<T>(actual: T, expected: T, msg: string): void {
  const ok = JSON.stringify(actual) === JSON.stringify(expected)
  if (ok) {
    passed++
  } else {
    failed++
    failures.push(`${msg} — expected ${JSON.stringify(expected)}, got ${JSON.stringify(actual)}`)
    console.error(`  ❌ ${msg}`)
    console.error(`     expected: ${JSON.stringify(expected)}`)
    console.error(`     got:      ${JSON.stringify(actual)}`)
  }
}

function group(name: string, fn: () => void): void {
  console.log(`\n${name}`)
  try {
    fn()
  } catch (e) {
    failed++
    failures.push(`${name} threw: ${(e as Error).message}`)
    console.error(`  ❌ group threw:`, e)
  }
}

// ==================== Tests start here ====================

group('State defaults', () => {
  const state = createDefaultState()
  assert(state.screen === 'home', 'default screen is home')
  assert(state.intervals.eyes === 20, 'default eyes interval')
  assert(state.streak === 0, 'default streak is 0')
})

group('Date helpers', () => {
  const t = todayStr()
  assert(/^\d{4}-\d{2}-\d{2}$/.test(t), 'todayStr is YYYY-MM-DD')
})

group('Streak logic — first use', () => {
  const state = createDefaultState()
  state.lastDate = ''
  updateStreak(state)
  assert(state.streak === 0, 'first use: streak starts at 0')
})

group('Streak logic — same day', () => {
  const state = createDefaultState()
  state.lastDate = todayStr()
  state.streak = 5
  updateStreak(state)
  assert(state.streak === 5, 'same day: streak unchanged')
})

// ... 100+ more tests ...

// ==================== Summary ====================

console.log(`\n${'═'.repeat(50)}`)
if (failed === 0) {
  console.log(`✓ ${passed} tests passed`)
} else {
  console.error(`✗ ${failed} failed, ${passed} passed`)
  for (const f of failures) console.error(`  - ${f}`)
  process.exit(1)
}
```

## Why this pattern (and not Jest)

- **Zero setup**: no `jest.config.js`, no `ts-jest`, no `@types/jest`, no `npm test` that takes 10 seconds to boot
- **Fast**: `npx tsx src/test-events.ts` runs in <500 ms for 100+ tests
- **No mocking framework needed**: most G2 app logic is pure functions on state — you don't need mocks, you need a plain-object input and an assertion on output
- **Portable**: every Node setup can run it; no framework lock-in
- **Readable**: entire test suite in one file, greppable

For apps that absolutely need a mocking framework (complex async flows, multi-module dependencies), upgrade to Vitest later. But start here.

## Exclude from tsconfig

Add to `tsconfig.json` so the test file doesn't pollute your production build:

```json
{
  "compilerOptions": { /* ... */ },
  "include": ["src"],
  "exclude": ["src/test-*.ts"]
}
```

## Add to package.json

```json
{
  "scripts": {
    "test": "tsx src/test-events.ts"
  }
}
```

You need `tsx` installed — `npm install --save-dev tsx`.

## What to test

Prioritize **pure logic on state** — these are the bugs that cost you the most:

### State transitions
```typescript
group('Screen transitions', () => {
  const state = createDefaultState()
  state.screen = 'home'

  // Simulate a "tap on home" action
  state.screen = 'settings'
  state.settingsCursor = 0

  assert(state.screen === 'settings', 'tap on home → settings')

  // Simulate a "tap on settings item"
  state.screen = 'settings_intervals'
  assert(state.screen === 'settings_intervals', 'select intervals')

  // Double-tap → home
  state.screen = 'home'
  assert(state.screen === 'home', 'double-tap → home')
})
```

### Timer / countdown math
```typescript
group('formatCountdown', () => {
  assert(formatCountdown(20) === '00:20', '20s')
  assert(formatCountdown(0) === '00:00', '0s')
  assert(formatCountdown(65) === '01:05', '65s')
})
```

### Streak across date boundaries
```typescript
group('Streak — consecutive days', () => {
  const state = createDefaultState()
  const yesterday = new Date()
  yesterday.setDate(yesterday.getDate() - 1)
  state.lastDate = yesterday.toISOString().slice(0, 10)
  state.streak = 5

  updateStreak(state)
  assert(state.streak === 5, 'yesterday: streak preserved')
})

group('Streak — skipped day', () => {
  const state = createDefaultState()
  const twoDaysAgo = new Date()
  twoDaysAgo.setDate(twoDaysAgo.getDate() - 2)
  state.lastDate = twoDaysAgo.toISOString().slice(0, 10)
  state.streak = 5

  updateStreak(state)
  assert(state.streak === 0, 'skipped day: streak resets')
})
```

### Event parsing (the critical one)

Replicate the parser inline in the test and verify all the edge cases:

```typescript
group('Event parsing — sysEvent fallback (CRITICAL)', () => {
  // Simulate what real hardware sends
  const event = {
    sysEvent: { eventType: 0, containerName: 'main', containerID: 0 },
  }
  const parsed = parseEvent(event)
  assert(parsed !== null, 'sysEvent parsed')
  assert(parsed?.action === 'click', 'sysEvent → click')
})

group('Event parsing — eventType 0 → undefined', () => {
  // Simulate JSON stripping the 0
  const event = {
    textEvent: { containerName: 'main', containerID: 0 }, // no eventType!
  }
  const parsed = parseEvent(event)
  assert(parsed?.action === 'click', 'undefined eventType → click')
})
```

### i18n coverage
```typescript
group('i18n coverage', () => {
  const locales = ['en', 'pt', 'es']
  const keys = ['welcome', 'settings', 'about', /* ... */]

  for (const locale of locales) {
    for (const key of keys) {
      const value = t(locale, key)
      assert(typeof value === 'string' && value.length > 0, `${locale}.${key} not empty`)
    }
  }
})
```

### Pure domain functions
```typescript
// Haversine distance
group('Haversine distance', () => {
  const d = calculateDistance(40.7128, -74.0060, 40.7128, -74.0060)
  assertEq(Math.round(d), 0, 'same point → 0m')

  const d2 = calculateDistance(40.7128, -74.0060, 40.7580, -73.9855)
  assert(d2 > 4000 && d2 < 6000, 'NYC → Central Park ~5km')
})
```

## What NOT to test with this pattern

- **Bridge calls** — you'd need to mock `EvenAppBridge` which gets complex. Do this in integration tests on the simulator.
- **React rendering** — if you have React UI, use `@testing-library/react` separately.
- **Network requests** — use Vitest or a real test framework with `fetch` mocking.
- **Real device I/O** — obviously.

For a G2 app, 80% of the bugs live in state/logic. This pattern catches 80% of the bugs with 20% of the setup.

## CI integration

Run the tests automatically:

```yaml
# .github/workflows/test.yml
name: Test
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm test
```

## Example: 107 tests in one file

See [`eyefit-g2/src/test-events.ts`](https://github.com/aleapc/eyefit-g2/blob/master/src/test-events.ts) for a complete example with 107 passing tests covering state, transitions, IMU tracking, event parsing, and i18n coverage.
