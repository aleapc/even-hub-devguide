# Reddit Post Drafts

Ready-to-copy posts for sharing the dev guide on Reddit. Choose the appropriate subreddit.

---

## r/smartglasses — long-form developer post

**Title:** Shipped 4 apps for Even Realities G2 — open-sourced them all and documented every SDK quirk I hit

**Body:**

I spent the last few weeks shipping four production apps on the Even Realities Even Hub platform for the G2 smart glasses. Since the docs are thin and the SDK has a handful of silent-failure bugs I couldn't find anyone else writing about, I put everything on GitHub and wrote a guide that consolidates what I learned.

**The apps** (all MIT, all on GitHub as `github.com/aleapc/<name>`):

- **eyefit-g2** — eye exercise coach with pixel art character demonstrating each exercise, IMU head tracking to verify rotation movements, and lifecycle-aware timers
- **hunter-g2** — place discovery app using Overpass/OSM with 27 pixel art category icons, offline cache, walking route guidance via OSRM
- **speechcoach-g2** — live speech pace coaching. This one evolved into a distributed architecture with a Node backend doing real Whisper STT so we could show actual WPM and filler-word counts in real time
- **breakmate-g2** — health reminders with an animated pixel art character that walks across the display. Paired with a React Native companion app using expo-notifications for scheduled reminders, since the G2 SDK has no background/push API

**The guide**: `github.com/aleapc/even-hub-devguide`

It documents 24 undocumented SDK quirks I had to figure out — things like "tap events arrive as `sysEvent` on real hardware but `textEvent` in the simulator" (this one cost me a day), "`canvas.toDataURL` PNGs are silently rejected by the image container — you need upng-js with 16-color indexed PNGs", "there is no background API so reminder-style apps need an Expo companion that relies on the native Even app's notification forwarding", etc.

Plus full recipes for:
- Event handling (drop-in template with all the workarounds)
- Pixel art pipeline (upng-js + sprite definitions + the exact `createStartUpPage → rebuild → 100ms delay → updateImageRawData` sequence)
- Lifecycle management (foreground/background/device status/graceful shutdown)
- Mobile companion pattern (expo-notifications forwarding to glasses)
- Distributed backend pattern (Express + SSE for heavy work like Whisper STT)
- Dev workflow (`evenhub qr` hot reload vs `.ehpk` upload)
- Testing (309 tests across the four apps using a lightweight `tsx`-runnable pattern, no Jest setup)
- Error telemetry (Cloudflare Worker + client pattern)

The guide credits everyone in the G2 community whose repos I learned from — BxNxM's even-dev, nickustinov's notes, fabioglimb's even-toolkit, kqb's openclaw HUD, sam-siavoshian's claude-code-g2, i-soxi's protocol reverse-engineering. If your work is on G2 and I missed you, please open a PR to add yourself.

Happy to answer questions about any of the specific quirks or patterns. Would love this to become a community-maintained reference rather than a personal project.

---

## r/EvenRealities — shorter community post

**Title:** 4 open-source G2 apps + a dev guide with all the SDK quirks nobody documents

**Body:**

Just open-sourced four apps I built for my G2: an eye exercise coach, a place finder, a speech pace analyzer, and a health reminder companion with an animated pixel art character. Plus a developer guide that documents 24 SDK quirks and the working patterns for pixel art, mobile companion reminders, and distributed STT backends.

`github.com/aleapc/even-hub-devguide` — the guide  
`github.com/aleapc/eyefit-g2`, `hunter-g2`, `speechcoach-g2`, `breakmate-g2` — the apps

Everything MIT licensed. Happy to answer questions.

---

## r/webdev — angle: "I built for smart glasses and it broke my assumptions"

**Title:** Show r/webdev: I shipped 4 smart glasses apps in plain TypeScript/React and documented everything

**Body:**

Smart glasses apps for Even Realities G2 are just standard web apps that run in a WebView on your phone and render to the glasses via BLE — Vite, TypeScript, React on the phone UI, optional backend services for heavy work. The SDK has no notification/background API, so I had to build reminder-style apps with a companion Expo app that forwards local notifications to the glasses via the native OS bridge. I also discovered a handful of silent SDK bugs that break apps on real hardware but work in the simulator (event types, image formats, container sequencing).

Open sourced all four apps plus a dev guide with 24 documented quirks:

github.com/aleapc/even-hub-devguide

Would love feedback from anyone who's built for unusual form factors — what patterns have you found for display-constrained, input-limited environments?
