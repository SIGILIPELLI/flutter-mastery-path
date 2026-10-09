---
description: "Level 1 of Flutter: the engine and widgets, project setup, composition, constraints, state, forms, lists and keys, theming, assets and a tested project."
---

# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: build small Flutter screens confidently — and understand *why* they lay out and update the way they do, so the
error messages make sense when they appear.

Three ideas carry this level:

1. **Everything is a widget, and widgets are cheap.** UI is a tree of small, immutable descriptions rebuilt whenever
   something changes. You compose; you don't mutate views.
2. **Constraints go down, sizes go up, the parent sets the position.** Almost every layout puzzle — overflow stripes,
   unbounded height errors, a `Row` that won't shrink — follows from that one rule.
3. **State lives in `State`.** `setState` marks a widget dirty; the next frame rebuilds it. Where you put state decides
   what rebuilds.

## Modules

1. [What Flutter Is: Engine, Widgets & One Codebase](01-what-flutter-is.md) — the engine, the framework, the platforms, and what "draws its own pixels" means
2. [Setup, flutter create & Project Anatomy](02-setup-first-app.md) — the toolchain, `flutter doctor`, the generated project, hot reload vs restart
3. [Widgets, build() & Composition](03-widgets-composition.md) — stateless widgets, composition over inheritance, `const`
4. [Layout: Constraints Go Down, Sizes Go Up](04-layout-constraints.md) — `Row`/`Column`/`Expanded`/`Flexible`, the overflow and unbounded errors explained
5. [StatefulWidget, setState & the State Lifecycle](05-stateful-widgets.md) — `initState`, `dispose`, `didUpdateWidget`, and what `setState` really does
6. [Text Input, Forms & Validation](06-forms-input.md) — controllers, `Form` and validators, focus, keyboard types
7. [Lists, Scrolling & Keys](07-lists-keys.md) — `ListView.builder`, slivers in brief, and why keys matter when items move
8. [Material 3 Theming](08-theming.md) — `ColorScheme.fromSeed`, text themes, dark mode, component themes
9. [Assets, Images & Fonts](09-assets.md) — `pubspec.yaml` assets, resolution variants, custom fonts, verifying what loads
10. [Project — A Tip Splitter with Widget Tests](10-project-tip-splitter.md) — integer-cents money logic, a responsive form, and widget tests for every control

## What you need before starting

- **Dart basics**: variables, functions, classes, null safety, `async`/`await`. The
  [Dart Mastery Path](https://sigilipelli.github.io/dart-mastery-path/) covers all of it.
- **Flutter SDK installed** (lesson 02 walks through `flutter doctor`). You can follow most of this level with only the SDK:
  widget tests run on your computer without a phone, emulator or simulator.

## How the examples were checked

Examples were run with **Flutter 3.44.8 (stable)** and **Dart 3.12.2**, mostly as widget tests with `flutter test`, whose
printed output is shown as it appeared. Widget tests render real widgets on a virtual 800×600 screen with a fake clock,
which is why they're deterministic.

!!! note "What wasn't run"
    The machine used to write this course couldn't build for Android (SDK licenses not accepted) or iOS (no Xcode), and had no device attached, so nothing in this level was run
    on a phone or simulator. Where a lesson describes device behaviour (hot reload on a device, platform look and feel), it's
    described from the documented behaviour and says so.
