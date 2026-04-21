# Twitter / X Thread (optional)

Ready-to-post thread if you want to share on Twitter/X. Tag `@EvenRealities` for visibility.

## Thread

**Tweet 1 (hook)**
> Shipped 4 apps for @EvenRealities G2 smart glasses and open-sourced everything, including a guide documenting 24 SDK quirks nobody talks about.
>
> If you're building on the Even Hub, this will save you days of debugging 👇
>
> github.com/aleapc/even-hub-devguide

**Tweet 2 (the one everyone hits)**
> The worst bug I hit: tap events arrive as `sysEvent` on real hardware but `textEvent` in the simulator.
>
> Your app works perfectly in the simulator. You ship it. You put it on real glasses. Zero input works. No error.
>
> Lost a full day to this one.

**Tweet 3 (pixel art)**
> The second worst: `canvas.toDataURL('image/png')` produces PNGs the G2's `imageToGray4` silently rejects.
>
> You need upng-js with 16-color indexed PNGs, manually quantized to 16 greyscale levels.
>
> Credit to @fabioglimb for the recipe (even-toolkit).

**Tweet 4 (architectural limit)**
> There's no background API. Your app must be in the foreground to render anything — timers fire but `rebuildPageContainer` calls are silently dropped.
>
> If you're building a reminder app, you need a companion mobile app that forwards notifications via the native Even app.

**Tweet 5 (the apps)**
> The four apps:
>
> 🔵 eyefit-g2 — eye exercise coach with IMU head tracking
> 🟢 hunter-g2 — place discovery with pixel art icons + OSRM routes
> 🟡 speechcoach-g2 — live speech coach with real Whisper STT
> 🟠 breakmate-g2 — animated pixel art character for health reminders
>
> All MIT, all linked from the guide.

**Tweet 6 (community credits)**
> This wouldn't exist without the G2 community. The guide credits @BxNxM (even-dev), nickustinov, @fabioglimb, kqb, sam-siavoshian, i-soxi.
>
> If you have a G2 project and I missed you, please open a PR.

**Tweet 7 (call to action)**
> Questions, corrections, or your own quirks to document? Open an issue or PR.
>
> Want this to be a community-maintained reference rather than a personal project.
>
> github.com/aleapc/even-hub-devguide
