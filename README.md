# Even Hub Case Studies — architectural patterns from 7 production G2 apps

> Case studies and architectural notes from shipping **7 production apps** on the Even Realities Even Hub platform for the G2 smart glasses.

This repo is **not** a general SDK reference or UI library — those already exist (see below). What's here is the stuff that tends to live *outside* the glasses themselves: how to pair the G2 with a companion mobile app, how to offload work to a backend, how to instrument errors from a WebView, how to produce images the SDK will actually render, and how to test pure logic without a Jest setup. Plus short case studies of the 7 apps where these patterns are exercised in production.

---

## Canonical references

Before anything here, look at these. They are the references this repo **complements** rather than duplicates.

| Repo | What it is |
|---|---|
| [`nickustinov/even-g2-notes`](https://github.com/nickustinov/even-g2-notes) | Community documentation bible — the go-to SDK reference, event catalog, and protocol notes. |
| [`fabioglimb/even-toolkit`](https://github.com/fabioglimb/even-toolkit) | UI library and design system. Pioneered the working pixel-art recipe with `upng-js`. |
| [`even-realities/evenhub-templates`](https://github.com/even-realities/evenhub-templates) | Official starter templates from Even Realities. |
| [`even-realities/everything-evenhub`](https://github.com/even-realities/everything-evenhub) | Official Claude Code plugin for G2 development. |

If you're new to G2 development, start with those. Come back here when you need a companion app, a backend, telemetry, or want to see how a full app is wired end-to-end.

---

## What's here

Patterns beyond the glass, all learned while shipping the apps listed below:

- [`docs/mobile-companion.md`](docs/mobile-companion.md) — Expo app + `expo-notifications` forwarding to the G2 via Even's native notification bridge. How to build reminder-style apps without the G2 app staying open.
- [`docs/distributed-backend.md`](docs/distributed-backend.md) — Express + SSE pattern for offloading work. Used by Speech Coach for real Whisper STT.
- [`docs/telemetry.md`](docs/telemetry.md) — Cloudflare Worker + KV for remote error reporting from WebView apps.
- [`docs/testing.md`](docs/testing.md) — state/logic test pattern using `tsx` directly, no Jest setup.
- [`docs/pixel-art.md`](docs/pixel-art.md) — sprite → 16-level greyscale → indexed PNG with `upng-js`, and how it hooks into `updateImageRawData`.
- [`docs/sdk-quirks.md`](docs/sdk-quirks.md) — catalog of quirks we hit in production (sysEvent vs textEvent, `eventType: 0` normalization, image size limits, etc.).
- [`docs/event-handling.md`](docs/event-handling.md) — drop-in `parseEvent` + handler skeleton.
- [`docs/lifecycle.md`](docs/lifecycle.md) — foreground/background, device info, battery awareness, graceful shutdown.
- [`docs/dev-workflow.md`](docs/dev-workflow.md) — hot reload with `evenhub qr`, simulator vs hardware gotchas, debug patterns.
- [`docs/getting-started.md`](docs/getting-started.md) — minimum viable app scaffold.

---

## Case studies — 7 production apps

Each repo is open source and its own case study.

| App | What it explores |
|---|---|
| [`hunter-g2`](https://github.com/aleapc/hunter-g2) | Place discovery with Overpass + Serper fetch, offline cache, OSRM walking routes, pixel-art category icons. |
| [`eyefit-g2`](https://github.com/aleapc/eyefit-g2) | Eye exercise guidance with IMU head tracking, pixel-art character, lifecycle-aware session persistence. |
| [`speechcoach-g2`](https://github.com/aleapc/speechcoach-g2) | Live speech pace coaching — audio capture → backend Whisper STT over SSE, VU meter, pixel-art mascot. |
| [`glance-g2`](https://github.com/aleapc/glance-g2) | Personal dashboard aggregator — multi-source data on a minimal display. |
| [`whatsapp-g2`](https://github.com/aleapc/whatsapp-g2) | WhatsApp messages from a Win11 bridge + voice reply via Whisper STT. |
| [`storywalk-g2`](https://github.com/aleapc/storywalk-g2) | Contextual tourism / running companion — GPS-tracked POI storytelling. |
| [`breakmate-g2`](https://github.com/aleapc/breakmate-g2) | Health reminders — animated pixel-art character, multi-frame walking animation, paired with an Expo companion. |

Companion repos referenced from the patterns above:

- [`breakmate-mobile`](https://github.com/aleapc/breakmate-mobile) and [`eyefit-mobile`](https://github.com/aleapc/eyefit-mobile) — Expo companions with `expo-notifications`.
- [`speechcoach-backend`](https://github.com/aleapc/speechcoach-backend) — Node + Express + SSE backend for STT.
- [`g2-telemetry-worker`](https://github.com/aleapc/g2-telemetry-worker) — Cloudflare Worker error endpoint.

---

## Submission compliance

All 7 apps received the same submission-compliance fix to call `shutDownPageContainer(1)` on the double-tap root (replacing the earlier `bridge.exit` / `shutDownPageContainer(0)` paths that the review flagged). The fix PRs, one per repo:

- [hunter-g2#1](https://github.com/aleapc/hunter-g2/pull/1)
- [eyefit-g2#1](https://github.com/aleapc/eyefit-g2/pull/1)
- [speechcoach-g2#1](https://github.com/aleapc/speechcoach-g2/pull/1)
- [glance-g2#1](https://github.com/aleapc/glance-g2/pull/1)
- [whatsapp-g2#1](https://github.com/aleapc/whatsapp-g2/pull/1)
- [storywalk-g2#1](https://github.com/aleapc/storywalk-g2/pull/1)
- [breakmate-g2#1](https://github.com/aleapc/breakmate-g2/pull/1)

If you hit the same submission rejection, those PRs show the minimal diff.

---

## Contributing

PRs welcome — especially corrections, counter-examples, or new patterns we haven't documented. See [CONTRIBUTING.md](CONTRIBUTING.md).

For SDK reference material, please contribute to [`nickustinov/even-g2-notes`](https://github.com/nickustinov/even-g2-notes) instead. For UI components, see [`fabioglimb/even-toolkit`](https://github.com/fabioglimb/even-toolkit). This repo stays focused on architectural patterns and case studies.

---

## License

MIT. See [LICENSE](LICENSE). Use freely, share freely.

## Disclaimer

Not affiliated with Even Realities. Community project documenting what actually worked in production as of SDK v0.0.9.
