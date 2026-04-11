# Getting Started with Even Hub Development

## Prerequisites

- Node.js 20+ (required by the SDK)
- A code editor with TypeScript support
- (Optional but recommended) An Even Realities G2 glasses pair for real hardware testing
- The Even Realities iOS or Android app on your phone (for sideloading via QR)

## Step 1: Create a new project

The fastest start is Vite with a TypeScript template:

```bash
npm create vite@latest my-g2-app -- --template vanilla-ts
cd my-g2-app
```

Or if you want React for the phone UI:

```bash
npm create vite@latest my-g2-app -- --template react-ts
cd my-g2-app
```

## Step 2: Install the Even Realities packages

```bash
npm install @evenrealities/even_hub_sdk
npm install --save-dev @evenrealities/evenhub-cli @evenrealities/evenhub-simulator
```

Pin the SDK version — APIs still change between patches:

```json
{
  "dependencies": {
    "@evenrealities/even_hub_sdk": "^0.0.9"
  }
}
```

## Step 3: Create `app.json` manifest

At the project root:

```json
{
  "package_id": "com.yourname.myapp",
  "edition": "202601",
  "name": "My App",
  "version": "0.1.0",
  "min_app_version": "0.1.0",
  "min_sdk_version": "0.0.9",
  "tagline": "Short description (50 chars max)",
  "description": "Longer description",
  "author": "Your Name",
  "entrypoint": "index.html",
  "supported_languages": ["en"],
  "permissions": []
}
```

**Note**: `supported_languages` only accepts `["en", "es"]` — Portuguese is NOT accepted here, even though your app can absolutely render Portuguese text. Handle PT internally if you need it.

## Step 4: Write your entry point

`src/main.ts`:

```typescript
import {
  waitForEvenAppBridge,
  TextContainerProperty,
  CreateStartUpPageContainer,
} from '@evenrealities/even_hub_sdk'

async function init() {
  // CRITICAL: always use waitForEvenAppBridge(), never `new EvenAppBridge()`
  // (the constructor is private in the published SDK)
  const bridge = await waitForEvenAppBridge()

  // Create the initial page with ONE text container
  const text = new TextContainerProperty({
    xPosition: 4,
    yPosition: 4,
    width: 568,          // 576 - 8 padding
    height: 280,         // 288 - 8 padding
    borderWidth: 0,
    borderColor: 0,
    borderRadius: 0,
    paddingLength: 8,
    containerID: 0,
    containerName: 'main',
    isEventCapture: 1,   // CRITICAL: exactly one container must have this
    content: 'Hello, G2!',
  })

  bridge.createStartUpPageContainer(new CreateStartUpPageContainer({
    containerTotalNum: 1,
    textObject: [text],
  }))

  // Subscribe to events (taps, scrolls, lifecycle)
  bridge.onEvenHubEvent((event) => {
    console.log('event:', event)
  })
}

init().catch(console.error)
```

## Step 5: Minimal `index.html`

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <title>My G2 App</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

## Step 6: Test in the simulator

```bash
npm run dev                        # Vite on :5173
npx evenhub-simulator http://localhost:5173
```

The simulator shows the 576×288 display. Click buttons on the simulator to trigger tap/scroll events.

**⚠️ CRITICAL WARNING**: the simulator does **not** send events the same way as real hardware. Your app can work perfectly in the simulator and be completely broken on real glasses. See [SDK quirk #2](../README.md#bug-2-real-hardware-sends-sysevent-simulator-sends-textevent). **Always test on real hardware before releasing.**

## Step 7: Test on real glasses (sideload via QR)

First login to the CLI:

```bash
npx evenhub login
```

Then generate a QR code pointing to your dev server:

```bash
# Find your LAN IP first:
#   Windows: ipconfig
#   macOS: ifconfig | grep inet
#   Linux: hostname -I

npx evenhub qr --ip 192.168.1.100 --port 5173
```

Scan the QR with the **Even Realities phone app** (Plugins → Scan QR). Your app loads on the glasses with full Vite hot reload.

## Step 8: Package for distribution

When you're ready to publish to the Hub:

```bash
npm run build                                      # produces dist/
npx evenhub pack app.json dist -o my-app.ehpk     # produces my-app.ehpk
```

Upload the `.ehpk` to [hub.evenrealities.com/hub](https://hub.evenrealities.com/hub) under your project's "Builds" tab.

## Recommended `package.json` scripts

```json
{
  "scripts": {
    "dev": "vite --host --port 5173",
    "build": "tsc && vite build",
    "pack": "npm run build && node -e \"const v=require('./package.json').version; require('child_process').execSync('npx @evenrealities/evenhub-cli pack app.json dist/ -o myapp-v'+v+'.ehpk', {stdio:'inherit'})\"",
    "sim": "npx evenhub-simulator http://localhost:5173",
    "qr": "npx evenhub qr --ip 192.168.1.100 --port 5173"
  }
}
```

Note the `pack` script: it embeds the version from `package.json` into the filename so you get `myapp-v0.3.1.ehpk` instead of overwriting a single file. Trust me, you want this.

## Next steps

- Read [SDK quirks](../README.md#sdk-quirks-and-workarounds) before writing any event handling code
- Copy the [event handling template](event-handling.md) as your starting point
- Study [one of the example projects](../README.md#example-projects) for your use case
