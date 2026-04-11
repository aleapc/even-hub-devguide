# Pixel Art Pipeline

How to render real pixel art images on the G2 display. **This is the only method that works reliably on real hardware.**

## TL;DR

- Use **[upng-js](https://github.com/photopea/UPNG.js/)** to produce 16-color indexed PNGs
- **NOT** `canvas.toDataURL('image/png')` — produces 24-bit RGBA that the host silently rejects
- Keep images under **200×100 px** (SDK docs say 288×144 but that's unreliable)
- Quantize greyscale to **16 levels**: `0, 17, 34, 51, ..., 255`
- Send via the exact sequence: dummy page → rebuild with image container → 100 ms delay → `updateImageRawData`

## Why canvas.toDataURL doesn't work

The G2 host runs `imageToGray4` to convert incoming pixel data to the hardware's 4-bit greyscale format. Empirically:
- 24-bit RGBA PNG from `canvas.toDataURL('image/png')`: host returns `imageToGray4Failed` silently
- 16-color indexed PNG from upng-js with pre-quantized greyscale values: **works**

Credit: this recipe was pioneered by [fabioglimb/even-toolkit](https://github.com/fabioglimb/even-toolkit). The entire community owes them one.

## Full recipe

### 1. Install upng-js

```bash
npm install upng-js
npm install --save-dev @types/upng-js
```

### 2. Define your sprite as ASCII

```typescript
// src/glasses/character.ts
const SPRITES: Record<string, string[]> = {
  idle: [
    '....ooooo...',
    '...oHHHHho..',
    '..oHHHHHHo..',
    '..ohhhhhho..',
    '..osseesso..',
    '..ossmmsso..',
    '...osssso...',
    '...orrro....',
    '..orrrrro...',
    '..orsrrro...',
    '..orrrrro...',
    '...opppo....',
    '..opppppo...',
    '..op...po...',
    '..ob...bo...',
    '..oo...oo...',
  ],
  // ... more poses
}
```

This is a 12×16 pixel sprite where each character is one pixel.

### 3. Define the greyscale color palette

Quantize each "color" to match the G2's 16-level greyscale output. The formula: `value * 17` gives the exact quantized level.

```typescript
const COLORS: Record<string, number> = {
  '.': 0,    // transparent → black
  'o': 34,   // dark outline (level 2)
  's': 204,  // skin (level 12)
  'S': 170,  // skin shadow (level 10)
  'h': 119,  // hat (level 7)
  'H': 170,  // hat light (level 10)
  'r': 102,  // shirt (level 6)
  'R': 68,   // shirt shadow (level 4)
  'p': 51,   // pants (level 3)
  'P': 34,   // pants shadow (level 2)
  'b': 85,   // shoes (level 5)
  'e': 51,   // eyes (level 3)
  'm': 68,   // mouth (level 4)
  'w': 255,  // white (level 15)
}
```

Each value is an exact multiple of 17 (17 × N where N = 0..15). This avoids re-quantization artifacts.

### 4. Scale up and encode to PNG

```typescript
import UPNG from 'upng-js'

const SPRITE_W = 12
const SPRITE_H = 16
const SCALE = 6  // 12*6=72, 16*6=96 — well under 200×100
export const IMG_W = SPRITE_W * SCALE
export const IMG_H = SPRITE_H * SCALE

/** Quantize a greyscale value to one of 16 levels */
function quantize(grey: number): number {
  const idx = Math.min(15, Math.round(grey / 17))
  return idx * 17
}

export function generateSpritePNG(poseName: string): number[] | null {
  try {
    const rows = SPRITES[poseName]
    if (!rows) return null

    // Build RGBA buffer by scaling each sprite pixel to SCALE × SCALE block
    const pixelCount = IMG_W * IMG_H
    const rgba = new Uint8Array(pixelCount * 4)

    for (let row = 0; row < SPRITE_H; row++) {
      const line = rows[row]
      for (let col = 0; col < SPRITE_W; col++) {
        const ch = line[col] ?? '.'
        const grey = quantize(COLORS[ch] ?? 0)

        // Fill SCALE × SCALE block
        for (let dy = 0; dy < SCALE; dy++) {
          for (let dx = 0; dx < SCALE; dx++) {
            const px = col * SCALE + dx
            const py = row * SCALE + dy
            const idx = (py * IMG_W + px) * 4
            rgba[idx] = grey
            rgba[idx + 1] = grey
            rgba[idx + 2] = grey
            rgba[idx + 3] = 255
          }
        }
      }
    }

    // Encode as 16-color indexed PNG
    const pngBuffer = UPNG.encode([rgba.buffer], IMG_W, IMG_H, 16)
    const pngBytes = new Uint8Array(pngBuffer)

    // Convert to number[] (SDK prefers this format)
    const result: number[] = new Array(pngBytes.length)
    for (let i = 0; i < pngBytes.length; i++) {
      result[i] = pngBytes[i]
    }
    return result
  } catch (e) {
    console.error('generateSpritePNG failed:', e)
    return null
  }
}
```

### 5. Display the image on the glasses

The **sequence matters**. Get any step wrong and nothing shows up.

```typescript
import {
  TextContainerProperty,
  ImageContainerProperty,
  ImageRawDataUpdate,
  CreateStartUpPageContainer,
  RebuildPageContainer,
} from '@evenrealities/even_hub_sdk'

let isFirstPage = true

async function showCharacterWithText(
  bridge: EvenAppBridge,
  poseName: string,
  text: string,
): Promise<void> {
  // STEP 1: Ensure a page exists. First call must be createStartUpPageContainer
  // with TEXT ONLY. You CANNOT include imageObject in the very first createStartUpPage call.
  if (isFirstPage) {
    const dummyText = new TextContainerProperty({
      xPosition: 4, yPosition: 4,
      width: 568, height: 280,
      borderWidth: 0, borderColor: 0, borderRadius: 0, paddingLength: 8,
      containerID: 0, containerName: 'main', isEventCapture: 1,
      content: 'Loading...',
    })

    bridge.createStartUpPageContainer(new CreateStartUpPageContainer({
      containerTotalNum: 1,
      textObject: [dummyText],
    }))
    isFirstPage = false

    // Small delay to let the page settle
    await new Promise(r => setTimeout(r, 100))
  }

  // STEP 2: Generate the PNG bytes
  const pngData = generateSpritePNG(poseName)
  if (!pngData || pngData.length === 0) {
    console.warn('Failed to generate PNG for pose:', poseName)
    return
  }

  // STEP 3: Rebuild the page declaring BOTH the text container AND the image container
  const textContainer = new TextContainerProperty({
    xPosition: 12 + IMG_W + 16,  // right of image
    yPosition: 4,
    width: 576 - (12 + IMG_W + 16) - 4,
    height: 280,
    borderWidth: 0, borderColor: 0, borderRadius: 0, paddingLength: 6,
    containerID: 0, containerName: 'main', isEventCapture: 1,
    content: text,
  })

  const imageContainer = new ImageContainerProperty({
    xPosition: 12,
    yPosition: Math.floor((288 - IMG_H) / 2),  // vertically centered
    width: IMG_W,
    height: IMG_H,
    containerID: 1,
    containerName: 'char',
  })

  bridge.rebuildPageContainer(new RebuildPageContainer({
    containerTotalNum: 2,
    textObject: [textContainer],
    imageObject: [imageContainer],
  }))

  // STEP 4: SHORT DELAY before pushing image data (critical!)
  await new Promise(r => setTimeout(r, 100))

  // STEP 5: Push the PNG bytes to the image container
  try {
    const result = await bridge.updateImageRawData(new ImageRawDataUpdate({
      containerID: 1,
      containerName: 'char',
      imageData: pngData,
    }))
    if (result && typeof result === 'string' && result !== 'success') {
      console.warn('updateImageRawData returned:', result)
    }
  } catch (e) {
    console.error('updateImageRawData failed:', e)
  }
}
```

### 6. Sequential-only sending (no concurrent calls)

`updateImageRawData` is **not safe to call concurrently**. Guard with a busy flag:

```typescript
let imageBusy = false

async function safeShowCharacter(bridge, pose, text) {
  if (imageBusy) return  // skip this call if another is in flight
  imageBusy = true
  try {
    await showCharacterWithText(bridge, pose, text)
  } finally {
    imageBusy = false
  }
}
```

### 7. Animation: push new PNG frames without rebuilding

Once the image container is declared on the page, you can push NEW frames (different poses) with just `updateImageRawData` — no `rebuildPageContainer` needed. This enables ~5 FPS animation without flicker.

```typescript
// After the initial rebuild+update, just push new frames:
async function pushFrame(bridge, poseName) {
  const pngData = generateSpritePNG(poseName)
  if (!pngData) return
  await bridge.updateImageRawData(new ImageRawDataUpdate({
    containerID: 1, containerName: 'char', imageData: pngData,
  }))
}

// Alternate two walking poses every 250 ms for a walking animation
let frame = 0
setInterval(() => {
  pushFrame(bridge, frame % 2 === 0 ? 'walk_1' : 'walk_2')
  frame++
}, 250)
```

**Cap animations at ~4-5 FPS** on real hardware. BLE bandwidth is limited.

## Size limits

- SDK docs: 288×144 pixels max
- Tested reliable: **200×100 pixels max**
- Our projects use 72×96 (12×16 sprite × 6x scale) — plenty of room, fast transmission

## Greyscale tips

- G2 displays in **green micro-LED** — all colors become shades of green
- `0` = off (invisible on the display)
- `255` = full brightness
- Stick to the 17-multiples palette (`0, 17, 34, ..., 255`) to avoid re-quantization
- High contrast works better than subtle gradients

## Common mistakes

❌ Using `canvas.toDataURL('image/png')` and then `atob` to byte array — 24-bit RGBA, silently rejected
❌ Including `imageObject` in the FIRST `createStartUpPageContainer` call — use rebuild
❌ Skipping the 100 ms delay between rebuild and updateImageRawData
❌ Calling `updateImageRawData` concurrently for multiple containers
❌ Sending images larger than 200×100
❌ Using colors that aren't multiples of 17 — get rounded, but waste bits

## Credits

The techniques in this doc were pioneered by **[fabioglimb/even-toolkit](https://github.com/fabioglimb/even-toolkit)**. We just distilled them and added the dos and don'ts we learned the hard way.
