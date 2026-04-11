# Contributing to the Even Hub Developer Guide

Thank you for considering a contribution! This guide exists to help **everyone** building G2 apps ship with confidence.

## Ways to contribute

### 🐛 Report a new SDK quirk

Found an undocumented bug or weird behavior? Open an issue or PR to [`docs/sdk-quirks.md`](docs/sdk-quirks.md). Include:

- Description (what you observed)
- SDK version (`@evenrealities/even_hub_sdk` version)
- Reproduction steps or a minimal code sample
- Workaround (if you figured one out)
- Whether it happens in the simulator, on hardware, or both

### ✅ Confirm or deny an existing quirk

Quirks get fixed in new SDK versions. If you can confirm that a documented bug is fixed, please update the doc to note the SDK version where it was resolved. Similarly, if a documented workaround stops working, please update it.

### 📖 Add a recipe

Struggled to figure something out? Write it up in `docs/`. Good recipes:
- Start with the problem statement
- Show complete, copy-pasteable code
- Explain **why** each step is needed (not just what)
- Link to any prior art you drew from

### 🌍 Translate the guide

Currently English-only. Translations welcome — drop them in `docs/<lang>/`. The README should link to available translations.

### 🔧 Fix something that's wrong

If you find an error, typo, or outdated info, just fix it. No ceremony.

### 🏷️ Add an example project

If you've built a G2 app and want it listed in the [Example projects](README.md#example-projects) table, open a PR adding it. Criteria:
- Open source (MIT/Apache/similar)
- Uses `@evenrealities/even_hub_sdk` (not a reverse-engineered BLE client)
- Actively maintained or stable
- Includes a README with setup instructions

## How to submit

1. Fork this repo
2. Create a branch: `git checkout -b fix/my-quirk-fix`
3. Make your change
4. Commit with a descriptive message
5. Push to your fork
6. Open a PR against `main`

## Code style

For TypeScript examples in the docs:

- Use the same patterns as the example projects (they're battle-tested)
- Prefer explicit types over `any`
- Include imports so code is copy-pasteable
- Target the latest stable SDK version
- Add comments for non-obvious workarounds (explain WHY, not just WHAT)

## Questions

Open a GitHub Discussion if you're not sure whether something belongs in the guide, or if you want to propose a larger restructuring.

## Code of conduct

Be kind. This is a community doc. People who contribute are giving their time for free.

## Credits

All contributors will be listed in the README's credits section. Small fixes don't need full attribution, but significant contributions (new docs, major rewrites) definitely do.
