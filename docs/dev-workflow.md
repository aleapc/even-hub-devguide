# Dev Workflow

Recommended tooling and patterns to iterate fast on G2 apps.

## Hot reload via `evenhub qr` (the professional path)

Instead of packaging a `.ehpk` and uploading it to the Hub on every change (painful), sideload via QR code and let Vite's hot reload handle updates.

### One-time setup

```bash
npm install -g @evenrealities/evenhub-cli
npx evenhub login
```

The login flow opens a browser for Even Realities auth.

### Per-session workflow

**Terminal 1**: dev server

```bash
cd your-project
npm run dev    # Vite on :5173
```

**Terminal 2**: QR sideload

```bash
# Find your LAN IP
#   Windows: ipconfig | findstr IPv4
#   macOS:   ifconfig | grep inet
#   Linux:   hostname -I

npx evenhub qr --ip 192.168.1.100 --port 5173
```

This prints a QR code in the terminal. Open the **Even Realities phone app** → **Plugins** → **Scan QR**. Your app loads on the glasses and Vite's HMR kicks in — every save triggers an instant reload on the glasses. No pack, no upload.

### Useful flags

- `-u, --url "http://192.168.1.100:5173"` — pass the full URL directly
- `-i, --ip` + `-p, --port` — more flexible equivalent
- `--path /index.html` — if your app isn't at the root
- `-e, --external` — open QR in an external image viewer (useful in WSL)
- `-s, --scale 3` — bigger QR (for external viewer)
- `--clear` — reset cached settings

### Time saved

| Action | Old (pack + upload) | New (QR + HMR) |
|---|---|---|
| Make a text change | 2-3 minutes | <1 second |
| Debug a visual bug | "rebuild, upload, relaunch" cycle | Instant reload |
| Iterate on a layout | 20 minutes for 10 tweaks | 2 minutes for 10 tweaks |

Once you've set up `evenhub qr`, you'll never go back.

## Simulator (`evenhub-simulator`)

For UI development without a real device:

```bash
npm run dev                                   # Vite
npx @evenrealities/evenhub-simulator http://localhost:5173
```

The simulator is great for **layout iteration** but has significant limitations. **Always test on real hardware** before releasing. Specifically:

- The simulator sends events via `textEvent` but real hardware sends via `sysEvent` — see [Quirk #2](sdk-quirks.md#quirk-2-real-hardware-sends-events-via-sysevent-simulator-via-textevent)
- The simulator doesn't emit lifecycle events (`FOREGROUND_ENTER`/`EXIT`)
- The simulator doesn't emit IMU or audio data
- The simulator's image size limits are more permissive than hardware

## Recommended `package.json` scripts

```json
{
  "scripts": {
    "dev": "vite --host --port 5173",
    "build": "tsc && vite build",
    "pack": "npm run build && node -e \"const v=require('./package.json').version; require('child_process').execSync('npx @evenrealities/evenhub-cli pack app.json dist/ -o myapp-v'+v+'.ehpk', {stdio:'inherit'})\"",
    "sim": "npx evenhub-simulator http://localhost:5173",
    "qr": "npx evenhub qr --ip $LAN_IP --port 5173",
    "test": "tsx src/test-events.ts"
  }
}
```

The **versioned pack** is important: it produces `myapp-v0.3.1.ehpk` instead of overwriting a single `myapp.ehpk`. You'll want this the first time you need to compare two versions.

## File structure (recommended)

Most of our apps follow this layout:

```
src/
├── main.ts                 # Entry point, bridge init, event handler setup
├── app.tsx                 # React phone UI (optional)
├── state.ts                # Types, defaults, helpers
├── i18n.ts                 # Translations
├── reminders.ts            # Domain logic (timers, business rules)
├── glasses/
│   ├── renderer.ts         # Screen dispatcher
│   ├── screens.ts          # Individual screen renderers
│   ├── events.ts           # Input handling (see event-handling.md)
│   ├── animation.ts        # Optional: animation helpers
│   ├── character.ts        # Optional: pixel art sprites (see pixel-art.md)
│   └── layout.ts           # Display constants
├── telemetry.ts            # Error reporting (optional)
└── test-events.ts          # Unit tests (see testing.md)
```

Not every app needs all of these, but this layout scales well.

## Debugging in the WebView

When your app loads in the Even Realities phone app, you can remote-debug the WebView:

**iOS**: Safari → Develop → (your iPhone) → (Even Realities app) → (your WebView instance)

**Android**: Chrome → `chrome://inspect` → find your device → inspect the Even Realities WebView

This gives you the full Chrome/Safari DevTools — Console, Network, Elements, Sources. Console logs from your G2 app appear here. Network requests show up. You can set breakpoints in your TypeScript source.

**Critical for debugging**: this is where you'll see the JavaScript errors that are otherwise invisible on the glasses. Make it part of your workflow.

## Common debug patterns

### Log every event to the console

Add this during development to see what real hardware is sending:

```typescript
bridge.onEvenHubEvent((event) => {
  console.log('EVENT:', JSON.stringify(event))
  // ... your real handler
})
```

Then open DevTools on the WebView and you'll see exactly what arrives.

### Fallback to text-only for debugging

When images don't render, fall back to a text message showing what went wrong:

```typescript
try {
  await showCharacterWithText(bridge, pose, text)
} catch (e) {
  // Show the error on the glasses so you know something failed
  showStatic(bridge, `IMAGE FAILED:\n${(e as Error).message.substring(0, 100)}`)
}
```

### Verbose logging with log levels

```typescript
const DEBUG = import.meta.env.DEV
function log(...args: unknown[]) {
  if (DEBUG) console.log('[MyApp]', ...args)
}
```

Gets stripped in production builds automatically.

## Versioning discipline

Bump your version **before** each build you want to share:

```bash
# Edit package.json: "version": "0.X.Y"
# Edit app.json:     "version": "0.X.Y"
npm run pack
```

You'll get `myapp-v0.X.Y.ehpk`. Upload that specific file to the Hub. Keep old `.ehpk` files around — they're invaluable when you need to regression-test.

Our apps use:
- **Patch bumps** (0.X.1 → 0.X.2): bug fixes, no behavior changes
- **Minor bumps** (0.X.0 → 0.Y.0): new features, breaking changes

Semver is loose for internal projects — the important thing is that each tested build has a unique version number.

## Next: read the [testing guide](testing.md)
