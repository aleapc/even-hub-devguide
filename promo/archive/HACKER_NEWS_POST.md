# Hacker News "Show HN" Post Draft

## Title options (pick one — title is everything on HN)

1. **Show HN: Even Hub Developer Guide — 24 SDK quirks from shipping 4 smart glasses apps**
2. **Show HN: I shipped 4 apps for Even Realities G2 glasses and documented every SDK bug I hit**
3. **Show HN: Developer guide for Even Realities G2 smart glasses — pixel art, STT, quirks**

*Recommended: option 1. It's specific, numerically anchored, and signals practical value.*

## URL

`https://github.com/aleapc/even-hub-devguide`

## Post body (HN allows a text post below the URL)

I spent the last few weeks shipping four production apps for the Even Realities G2 smart glasses on the Even Hub platform. The docs are thin, the SDK has a bunch of silent-failure bugs, and the simulator lies to you in ways that only show up when you test on real hardware. This guide is an attempt to consolidate everything I had to learn the hard way.

Highlights from the quirks list:

- Tap events arrive as `sysEvent` on real hardware but `textEvent` in the simulator. Naive parsers checking only `textEvent` ship "working" apps that have totally dead input on glasses. Lost a day to this.
- `eventType: 0` (CLICK_EVENT) serializes as `undefined` in the bridge's JSON path. `if (eventType === 0)` never matches. You need a normalizer at the parse boundary.
- `canvas.toDataURL('image/png')` produces 24-bit RGBA PNGs that the glasses' `imageToGray4` function silently rejects. You need `upng-js` with 16-color indexed PNGs, quantized to 16 greyscale levels. Max image size is also ~200×100, not the 288×144 the docs advertise.
- You cannot declare an image container in `createStartUpPageContainer` — it must be text-only first, then `rebuildPageContainer` with the image container declared, then a 100ms delay, then `updateImageRawData`. Skip any step, silent failure.
- The SDK has no background/push/scheduled-launch API at all. Your app must be in the foreground to render anything. Reminder-style apps require a companion mobile app using local notifications + the native Even app's notification forwarding.

Plus 19 more quirks and a full set of recipes: event handling template with all workarounds, pixel art pipeline with working upng-js code, lifecycle management, mobile companion pattern with Expo, distributed backend pattern (Express + SSE for heavy work like Whisper STT), dev workflow with `evenhub qr` hot reload, testing pattern, and a Cloudflare Worker for error telemetry.

The four apps are all open source and linked from the guide. One of them is an animated pixel art character that walks across the display delivering health reminders. Another does live speech pace coaching using a Node backend with real Whisper STT streaming.

Credits go to the G2 community whose reverse-engineering work made this possible — the guide links `BxNxM/even-dev` (simulator + 16 example apps), `nickustinov/even-g2-notes`, `fabioglimb/even-toolkit` (pioneered the pixel art recipe), `kqb/openclaw-g2-hud`, `sam-siavoshian/claude-code-g2`, and `i-soxi/even-g2-protocol`.

Would love feedback, PRs, and to hear from anyone else building for constrained-display wearables. If you've hit a quirk that isn't in the list, please open an issue — I want this to be a community-maintained reference.

## Submission tips

- Post Tuesday–Thursday, 8–10am EST for best visibility
- Be ready to answer questions in the comments within the first 2 hours (HN is front-loaded)
- Don't upvote your own post (rule violation, can get flagged)
- If it takes off, be responsive and factual in comments — HN engagement rewards substance
