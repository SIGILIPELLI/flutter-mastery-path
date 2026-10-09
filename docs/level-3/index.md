---
description: "Level 3 of Flutter: elements and keys, animations, CustomPainter, responsive layouts, platform channels, isolates, Bloc, a11y, i18n, offline-first."
---

# Level 3 · Advanced <span class="level-badge">Depth</span>

Goal: understand the framework underneath your widgets, and build the things that separate polished apps from prototypes —
custom graphics, motion, adaptive layouts, native integration, accessibility, and apps that work offline.

Three ideas carry this level:

1. **Elements are the real tree.** Widgets are recreated constantly; elements (and the `State` they hold) persist when type and
   key match. Keys, lost text fields and "deactivated widget" errors all follow from this.
2. **Do less per frame.** Animate render-object properties instead of rebuilding, repaint only what changed, and move heavy
   CPU work off the UI isolate.
3. **Design for every user and every condition** — screen readers and large text, other languages, wide windows, flaky
   networks.

## Modules

1. [Widgets, Elements & BuildContext](01-element-tree.md) — five experiments on when state is kept, lost or moved
2. [Animations: Implicit & Explicit](02-animations.md) — curves sampled frame by frame, staggered intervals, leak diagnostics
3. [CustomPainter & Drawing](03-custom-painter.md) — a sparkline with tested math, `shouldRepaint` vs layers, semantics, goldens
4. [Responsive & Adaptive Layouts](04-responsive.md) — window size classes, rail vs bar, list–detail, tested at three sizes
5. [Platform Channels](05-platform-channels.md) — MethodChannel and EventChannel, Kotlin and Swift handlers, the bytes on the wire
6. [Isolates & Heavy Work](06-isolates.md) — measuring a blocked UI isolate and fixing it with `Isolate.run`
7. [The Bloc Pattern](07-bloc.md) — Cubit vs Bloc, sealed states, debounced search, `bloc_test`
8. [Accessibility & Internationalization](08-a11y-i18n.md) — semantics, guideline tests, text scaling, gen-l10n with ICU plurals
9. [Integration Tests & Test Strategy](09-integration-tests.md) — what needs a device, how to run it, and a test pyramid that holds up
10. [Project — An Offline-First Habit Tracker](10-project-habit-tracker.md) — a persisted outbox, conflict resolution, honest sync status

## What you need before starting

- [Level 2](../level-2/index.md): routing, a state-management approach, async UI and widget testing.
- For lesson 05, reading knowledge of Kotlin or Swift helps; the [Kotlin](https://sigilipelli.github.io/kotlin-mastery-path/) and
  [Swift](https://sigilipelli.github.io/swift-mastery-path/) courses cover the languages.

## How the examples were checked

Examples were run with **Flutter 3.44.8** / **Dart 3.12.2**, plus **flutter_bloc 9.1.1**, **bloc_test 10.0.0** and **intl 0.20.2**
where used. Lesson 06's timings come from `dart run` on one Apple Silicon Mac and are labelled as such — they illustrate the
effect, they aren't benchmarks.

!!! note "What wasn't run"
    The Kotlin and Swift code in lesson 05 was not compiled (Android builds were blocked by unaccepted SDK licenses and Xcode wasn't installed on the course machine); its Dart side was
    tested against mock handlers. Lesson 09's integration test could not be run on a device for the same reason — the lesson
    shows the exact errors the attempts produced and what was verified instead.
