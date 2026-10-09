---
description: "Level 2 of Flutter: navigation and go_router, InheritedWidget, Provider, Riverpod, async UI, HTTP and JSON, persistence, testing and a reading-list app."
---

# Level 2 · Intermediate <span class="level-badge">Real Apps</span>

Goal: build multi-screen apps that load data, remember it, share state cleanly between widgets, and come with tests you
trust.

Three ideas carry this level:

1. **Navigation is state.** Imperative `push`/`pop` is fine for detours; anything addressable (deep links, the web) should be
   derived from a location with a router.
2. **Share state by subscription, not by passing it everywhere.** `InheritedWidget` is the mechanism; Provider and Riverpod are
   the ergonomic layers on top, and the skill is subscribing to exactly what each widget needs.
3. **Put I/O behind seams.** HTTP clients and storage are injected, so every screen can be tested with fakes — no network, no
   disk, no device.

## Modules

1. [Navigation: push, pop & Returning Results](01-navigator.md) — the route stack, awaiting results, `PopScope` for unsaved changes
2. [Declarative Routing with go_router](02-go-router.md) — nested routes, parameters, auth redirects, shell routes
3. [Lifting State & InheritedWidget](03-inherited-widget.md) — who rebuilds and why, measured
4. [Provider & ChangeNotifier](04-provider.md) — `watch`, `read` and `select`, with rebuild counts
5. [Riverpod](05-riverpod.md) — Notifier, AsyncNotifier, derived providers, overrides in tests
6. [Async UI: FutureBuilder & StreamBuilder](06-async-ui.md) — snapshot states, retry, and the future-in-build trap
7. [HTTP & JSON](07-http-json.md) — a typed client with validation, error mapping and `MockClient` tests
8. [Local Persistence](08-persistence.md) — shared_preferences, atomic JSON files, schema migration, when to use SQLite
9. [Testing: Unit, Widget & Golden Tests](09-testing.md) — table-driven tests, fakes with `Completer`, golden images and their failures
10. [Project — A Reading List App](10-project-reading-list.md) — go_router + Riverpod + a storage repository, with a full test suite

## What you need before starting

- [Level 1](../level-1/index.md), or equivalent experience: stateful widgets, layout, forms and lists.
- Comfort with Dart `Future`s and `Stream`s ([Dart Mastery Path](https://sigilipelli.github.io/dart-mastery-path/)).
- For lesson 07, the basics of HTTP and JSON APIs; the [REST API Mastery Path](https://sigilipelli.github.io/rest-api-mastery-path/)
  covers them.

## How the examples were checked

Every example was run with **Flutter 3.44.8** and **Dart 3.12.2** using **go_router 18.0.2**, **provider 6.1.5+1**,
**flutter_riverpod 3.4.3**, **http 1.6.0** and **shared_preferences 2.5.6**. Outputs are the actual `print` output of
`flutter test`, with the tool's per-test progress lines removed for readability. Several lessons include first attempts that
failed (a `setState` that returned a `Future`, a test expectation that was wrong) because the failures teach as much as the fixes.

!!! note "What wasn't run"
    No real network calls were made — HTTP examples use `MockClient` against a reserved example domain — and platform storage was
    replaced by in-memory fakes in tests. Nothing was run on a phone or simulator; lesson 08 and the project say exactly what
    that does and doesn't prove.
