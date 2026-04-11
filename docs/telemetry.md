# Error Telemetry

When your G2 app runs on real hardware with real users, bugs happen that you can't see. You need remote error reporting.

Our [g2-telemetry-worker](https://github.com/aleapc/g2-telemetry-worker) is a Cloudflare Worker that accepts error reports from G2 apps via a simple POST endpoint. All four of our apps use it.

## Architecture

```
[G2 app in WebView] → POST /report → [Cloudflare Worker] → [KV storage, 7-day TTL]
                                               ↓
                                     [GET /reports (admin)]
```

The worker:
- Accepts POST `/report` from any origin (CORS-enabled)
- Validates and stores each report in KV with a 7-day TTL
- Exposes GET `/reports?app={name}` behind an admin token for viewing
- Has simple in-memory rate limiting per IP
- Has a `/health` endpoint

## Client integration

Every G2 app drops in a `src/telemetry.ts` that:

1. Auto-captures `window.error` events
2. Auto-captures unhandled promise rejections
3. Exposes a `reportError()` function for manual reporting
4. Silently fails on network errors (telemetry must NEVER crash the app)

Minimal client:

```typescript
// src/telemetry.ts
const TELEMETRY_URL = 'https://g2-telemetry.your-subdomain.workers.dev'
const APP_NAME = 'myapp'        // change per app
const APP_VERSION = '0.1.0'     // bump with releases

export interface TelemetryContext {
  [key: string]: unknown
}

export async function reportError(
  message: string,
  stack?: string,
  context?: TelemetryContext,
): Promise<void> {
  try {
    await fetch(`${TELEMETRY_URL}/report`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        app: APP_NAME,
        version: APP_VERSION,
        message,
        stack,
        context,
        timestamp: Date.now(),
        userAgent: typeof navigator !== 'undefined' ? navigator.userAgent : undefined,
      }),
    })
  } catch {
    // Silent fail — telemetry must never crash the app
  }
}

// Auto-capture unhandled errors
if (typeof window !== 'undefined') {
  window.addEventListener('error', (event) => {
    reportError(event.message, event.error?.stack, {
      source: event.filename,
      line: event.lineno,
      column: event.colno,
    })
  })

  window.addEventListener('unhandledrejection', (event) => {
    reportError(
      `Unhandled promise rejection: ${event.reason?.message ?? event.reason}`,
      event.reason?.stack,
    )
  })
}
```

Import once at the top of your entry point:

```typescript
// src/main.ts
import './telemetry'  // self-initializes error listeners

import { waitForEvenAppBridge } from '@evenrealities/even_hub_sdk'
// ... rest of init
```

## Manual reporting for expected error paths

Use `reportError()` directly when you catch errors you'd like to know about:

```typescript
try {
  await bridge.updateImageRawData(new ImageRawDataUpdate({ /* ... */ }))
} catch (e) {
  // Expected during poor connectivity; log it but don't crash
  reportError('Image update failed', (e as Error).stack, {
    imageSize: pngData.length,
    containerID: 1,
  })
  // Fall back to text-only
  rebuildTextOnly(bridge, text)
}
```

## Deploying the worker

```bash
cd g2-telemetry-worker
npm install

# Create KV namespace
npx wrangler kv:namespace create TELEMETRY_KV
# Copy the ID into wrangler.toml

# Set admin token for /reports endpoint
npx wrangler secret put ADMIN_TOKEN

# Deploy
npx wrangler deploy
```

You'll get a URL like `https://g2-telemetry.your-subdomain.workers.dev`. Paste it into `TELEMETRY_URL` in each app's `src/telemetry.ts`.

## Viewing reports

```bash
curl -H "x-admin-token: $ADMIN_TOKEN" \
  "https://g2-telemetry.your-subdomain.workers.dev/reports?app=myapp"
```

Returns JSON array of recent reports. Pipe through `jq` or write a little HTML dashboard.

## Privacy notes

- Don't log user-identifiable data (email, name, location) in error reports
- Don't log authentication tokens
- Keep the retention short (7 days is plenty)
- Make the telemetry opt-out if you're publishing the app publicly

## Alternatives

If you don't want to self-host:

- **Sentry** — full-featured, free tier, overkill for this use case
- **LogRocket / Datadog** — heavier, $$$
- **Axiom** — log aggregation with a free tier
- **A Google Sheets webhook** — seriously, this works in a pinch

Our Cloudflare Worker approach is ~100 lines of code, free for typical usage, and sufficient for catching the weird bugs that only happen on real devices.

## Example

[`g2-telemetry-worker`](https://github.com/aleapc/g2-telemetry-worker) — complete implementation
[`src/telemetry.ts` in breakmate-g2](https://github.com/aleapc/breakmate-g2/blob/master/src/telemetry.ts) — client integration example
