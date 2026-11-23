---
title: "Flutter Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Flutter cheat sheet: 18 quick-reference tables from every level of the Flutter Mastery Path, each linked to its full lesson."
---

# Flutter Cheat Sheet

Quick-reference tables from **18 lessons** of the free Flutter Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · What Flutter Is: Engine, Widgets &amp; One Codebase](level-1/01-what-flutter-is.md)

*What this design buys and costs*

| | Consequence |
|---|---|
| Draws its own pixels | UI looks identical across platforms; no "works on Android, broken on iOS" layout bugs |
| Engine ships in the app | larger downloads than a pure-native app; startup has to load the engine |
| Everything is Dart | one language for UI and logic; full source of the framework is readable |
| Not native controls | platform look-and-feel must be chosen (Material, Cupertino, adaptive widgets); accessibility and text input go through Flutter's own bridges |
| Hot reload | very fast iteration in debug builds |

### [03 · Widgets, build() &amp; Composition](level-1/03-widgets-composition.md)

*The component hierarchy*

| Widget | Job |
|---|---|
| `Card` | Material surface with elevation and rounded corners |
| `Padding` | Insets its child |
| `Row` / `Column` | Lay children out horizontally / vertically |
| `Expanded` | Makes a `Row`/`Column` child take the remaining space |
| `SizedBox` | A fixed-size box — here, a 12-pixel gap |
| `CircleAvatar`, `Badge`, `Icon` | Material visuals |

### [05 · StatefulWidget, setState &amp; the State Lifecycle](level-1/05-stateful-widgets.md)

*The lifecycle, observed*

| Call | When | Use it for |
|---|---|---|
| `createState` | widget inserted at a new place | creating the `State` (nothing else) |
| `initState` | once, right after creation | one-time setup: controllers, timers, subscriptions |
| `didChangeDependencies` | after `initState`, and whenever an inherited widget it depends on changes | work that needs `Theme.of`, `MediaQuery.of`, etc. |
| `build` | every time the state is dirty or its parent rebuilds | describing UI — nothing else |
| `didUpdateWidget` | parent rebuilt with a new widget of the same type | reacting to changed configuration |
| `dispose` | permanently removed | cancel timers, close streams, dispose controllers |

### [07 · Lists, Scrolling &amp; Keys](level-1/07-lists-keys.md)

*Choosing a key*

| Key | Use when |
|---|---|
| `ValueKey(id)` | items have a stable unique value — a database id is ideal |
| `ObjectKey(obj)` | identity of the object itself is the identity of the row |
| `UniqueKey()` | force a *new* state every build (rarely what you want in lists) |
| `GlobalKey` | access a widget's state from elsewhere, or move a subtree across parents — expensive, use sparingly |

## Level 2 · Intermediate

### [04 · Provider &amp; ChangeNotifier](level-2/04-provider.md)

*Providing and consuming*

| Call | Subscribes? | Use it in | For |
|---|---|---|---|
| `context.watch<Cart>()` | yes, to every notification | `build` | widgets that display lots of the model |
| `context.select<Cart, T>((c) => ...)` | yes, but rebuilds only if the selected value changes (`==`) | `build` | widgets that need one derived value |
| `context.read<Cart>()` | no | callbacks (`onPressed`) or `build` when you only need methods | calling methods |

### [05 · Riverpod](level-2/05-riverpod.md)

*The provider types you'll actually use*

| Declaration | State | Typical use |
|---|---|---|
| `Provider<T>((ref) => ...)` | a computed value | dependencies (a repository, an HTTP client), derived data |
| `NotifierProvider<N, T>(N.new)` | synchronous `T`, changed by methods on `N` | UI state: filters, selection, form state |
| `AsyncNotifierProvider<N, T>(N.new)` | `AsyncValue<T>` loaded in `build()`, then changed by methods | data from a backend that the user edits |
| `FutureProvider<T>` / `StreamProvider<T>` | `AsyncValue<T>` | read-only async data |

### [06 · Async UI: FutureBuilder &amp; StreamBuilder](level-2/06-async-ui.md)

*Snapshots*

| Field | Meaning |
|---|---|
| `connectionState` | `none` (no future/stream), `waiting` (nothing yet), `active` (stream has emitted, still open), `done` |
| `hasData` / `data` | the latest value, if any |
| `hasError` / `error` / `stackTrace` | the error, if the future failed or the stream emitted one |

### [08 · Local Persistence](level-2/08-persistence.md)

| Data | Good fit |
|---|---|
| A few settings and flags | `shared_preferences` (key–value) |
| A document or small collection you load and save whole | a JSON file in the app's documents directory |
| Many records you query, filter, sort, page | SQLite (`sqflite`, or `drift` for a typed layer) or another embedded database |
| Secrets (tokens, keys) | platform keystores via `flutter_secure_storage` |
| Large binary files (images, downloads) | files on disk, with paths stored in one of the above |

### [09 · Testing: Unit, Widget &amp; Golden Tests](level-2/09-testing.md)

| Kind | Tests | Runs on | Speed |
|---|---|---|---|
| Unit (`test`) | a function or class, no widgets | the Dart VM | milliseconds |
| Widget (`testWidgets`) | a widget subtree, rendered headlessly | the Dart VM, fake clock | tens of ms |
| Golden (`matchesGoldenFile`) | a widget's *pixels* against a stored PNG | the Dart VM | tens of ms |
| Integration (`integration_test`) | the whole app on a device/emulator | a real device | seconds–minutes |

## Level 3 · Advanced

### [01 · Widgets, Elements &amp; BuildContext](level-3/01-element-tree.md)

*Three trees*

| Tree | Objects | Lifetime | Job |
|---|---|---|---|
| Widget | `Text`, `Padding`, your widgets | rebuilt constantly, immutable, cheap | *configuration* |
| Element | `StatelessElement`, `StatefulElement`, … | long-lived, mutable | *identity and position*; holds `State` |
| Render | `RenderParagraph`, `RenderPadding`, … | long-lived, created by some elements | layout, paint, hit-testing |

### [02 · Animations: Implicit &amp; Explicit](level-3/02-animations.md)

*Choosing*

| Need | Use |
|---|---|
| A property should animate when state changes | `AnimatedFoo` / `TweenAnimationBuilder` |
| Swap one widget for another with a transition | `AnimatedSwitcher` |
| Shared element between routes | `Hero` |
| Loop, sequence, scrub, or drive from a gesture | `AnimationController` + `*Transition` / `AnimatedBuilder` |
| Physics (fling, spring) | `controller.animateWith(SpringSimulation(...))` |
| Designer-made vector animations | packages such as `rive` or `lottie` |

### [05 · Platform Channels](level-3/05-platform-channels.md)

*Three kinds of channel*

| Channel | Shape | Use for |
|---|---|---|
| `MethodChannel` | request → one reply (a `Future`) | "get battery level", "start scan", "save to keychain" |
| `EventChannel` | subscribe → stream of events | sensor readings, connectivity changes, download progress |
| `BasicMessageChannel` | raw messages either way | custom protocols, rarely needed directly |

### [07 · The Bloc Pattern](level-3/07-bloc.md)

*Cubit and Bloc*

| | Input | Good for |
|---|---|---|
| `Cubit<State>` | method calls (`cubit.toggle()`) that `emit` new states | simple state: a theme, a counter, a form toggle |
| `Bloc<Event, State>` | event objects (`bloc.add(QueryChanged('he'))`) handled by `on<Event>` | flows where *how* events are processed matters: debouncing, cancelling, ordering, logging every input |

### [09 · Integration Tests &amp; Test Strategy](level-3/09-integration-tests.md)

*A test strategy that holds up*

| Layer | What | How many | Example from this course |
|---|---|---|---|
| Unit | pure logic, models, parsing, notifiers | many | `passwordScore`, `calculateSplit`, `ReadingList` rules |
| Widget | screens and flows with fakes | many | signup form, reading-list add/move/rate/remove |
| Golden | appearance of key components | a few | strength meter, sparkline |
| Integration | real platform, end-to-end | a handful of critical journeys | add book → restart → still there |

## Level 4 · Master

### [01 · The Rendering Pipeline &amp; RenderObjects](level-4/01-rendering-pipeline.md)

*One frame, five phases*

| Phase | Who's involved | What happens |
|---|---|---|
| **Animate** | tickers | animation controllers advance ([Level 3 · 02](level-3/02-animations.md)) |
| **Build** | dirty elements | `build()` runs for elements marked by `setState` & co; widgets diffed against elements |
| **Layout** | render objects needing layout | constraints go down, sizes come up, parents position children |
| **Paint** | render objects needing paint | drawing commands recorded into layers |
| **Composite** + **semantics** | layer tree, semantics owner | layer tree sent to the engine for rasterization; accessibility tree updated |

### [04 · Configuration, Flavors &amp; --dart-define](level-4/04-flavors-config.md)

| Mechanism | Changes | Set with |
|---|---|---|
| **Dart defines** | values your Dart code reads at compile time (API URL, feature switches) | `--dart-define=KEY=value`, `--dart-define-from-file=config/dev.json` |
| **Flavors** | the *native* app: bundle/application id, app name, icon, signing, native SDK keys | Android `productFlavors`, iOS/macOS Xcode schemes; `--flavor dev` |

### [07 · Flutter on the Web &amp; Desktop](level-4/07-web-desktop.md)

*Web: two compilers, one app*

| | JavaScript build | `--wasm` build |
|---|---|---|
| Compiled app code | `main.dart.js`, 2,571,692 bytes (762,231 gzipped) | `main.dart.wasm`, 2,357,235 bytes (866,257 gzipped) + a 32,735-byte `main.dart.mjs` loader |
| Renderer | CanvasKit (Skia compiled to WebAssembly) | Skwasm where the browser supports WasmGC, falling back to the JS build otherwise |

### [10 · Capstone — A Production-Ready Expense Tracker](level-4/10-capstone.md)

*What it would take to ship*

| Gap | Where |
|---|---|
| Run on real devices; integration test for "add → restart → still there" with the real file store | [Level 3 · 09](level-3/09-integration-tests.md) |
| Budget as a user setting (persisted) instead of a constant | [Level 2 · 08](level-2/08-persistence.md) |
| Localized strings and currency per user locale | [Level 3 · 08](level-3/08-a11y-i18n.md) |
| Wide-screen layout (list + detail) for tablets and desktop | [Level 3 · 04](level-3/04-responsive.md) |
| CI with format/analyze/test gates | [Level 4 · 05](level-4/05-ci.md) |
| Flavors, signing, store listing, privacy declarations | [Level 4 · 04](level-4/04-flavors-config.md), [06](level-4/06-publishing.md) |
| Optional sync across devices | [Level 3 · 10](level-3/10-project-habit-tracker.md) |

---

Go deeper with the full [Flutter Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
