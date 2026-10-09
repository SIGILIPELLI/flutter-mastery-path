---
description: "Level 4 of Flutter: rendering pipeline, performance, architecture, flavors, CI, store publishing, web and desktop, FFI, upgrades and a capstone."
---

# Level 4 · Master <span class="level-badge">Production</span>

Goal: run Flutter like a professional — understand the pipeline well enough to reason about performance, structure an app so a
team can change it for years, and get it built, tested, configured and shipped.

Three ideas carry this level:

1. **Measure, then change.** Count rebuilds, layouts and paints; profile on devices; keep the numbers in tests so fixes don't rot.
2. **Boundaries are what make change cheap.** Layers with inward dependencies, a single composition root, and tests that enforce
   the rules.
3. **Shipping is a pipeline, not an event.** Formatting, analysis, tests, configuration, signing and upgrades — automated and
   repeated, small and often.

## Modules

1. [The Rendering Pipeline & RenderObjects](01-rendering-pipeline.md) — a custom `RenderBox`, and which changes trigger layout vs paint
2. [Performance: Rebuilds, Jank & DevTools](02-performance.md) — profile mode, measured fixes, and a list-building myth corrected
3. [App Architecture: Layers & Dependency Injection](03-architecture.md) — feature-first layers, a composition root, an architecture test
4. [Configuration, Flavors & --dart-define](04-flavors-config.md) — validated compile-time config, native flavors, secrets
5. [CI for Flutter](05-ci.md) — a local CI script, a real first run that failed, and a GitHub Actions workflow
6. [Building & Publishing to the Stores](06-publishing.md) — versioning, signing keys, App Bundles, TestFlight, privacy declarations
7. [Flutter on the Web & Desktop](07-web-desktop.md) — JS vs Wasm builds measured, tests in Chrome, `dart:io` pitfalls, desktop expectations
8. [Plugins & dart:ffi](08-plugins-ffi.md) — plugin anatomy, federated plugins, and C called from Dart for real
9. [Upgrades, Dependencies & Long-Lived Apps](09-upgrades.md) — `pub outdated`, `dart fix`, SDK pinning, an upgrade routine
10. [Capstone — A Production-Ready Expense Tracker](10-capstone.md) — every layer, optimistic updates with rollback, undo, a11y, goldens, 15 tests

## What you need before starting

- [Level 3](../level-3/index.md), or real-world Flutter experience.
- For lessons 05 and 06, a GitHub account and familiarity with Git helps — see the
  [GitHub & Git Mastery Path](https://sigilipelli.github.io/github-mastery-path/).

## How the examples were checked

Run with **Flutter 3.44.8** / **Dart 3.12.2**. The CI script, web builds (JavaScript and Wasm), `flutter test --platform chrome`,
`keytool`, `dart fix`, `pub outdated` and the C library for FFI (built with the system `clang`) were all run, and their output is
shown. Timings and file sizes are from one machine and one app.

!!! note "What wasn't run"
    Android, iOS and desktop builds, store uploads, profile-mode DevTools sessions and the GitHub Actions workflow were not run:
    on the course machine Android builds were blocked by unaccepted SDK licenses, Xcode wasn't installed, and no store accounts
    were used. Those parts are described from the documented
    process, are labelled in each lesson, and should be verified against current official documentation before you rely on them.
