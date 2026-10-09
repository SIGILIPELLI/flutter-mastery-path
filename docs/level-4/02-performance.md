---
description: "Find and fix Flutter jank: profile mode and DevTools, measured rebuild counts, lazy lists, localized setState, const and render-level animation."
---

# 02 · Performance: Rebuilds, Jank & DevTools

Users don't notice milliseconds; they notice **dropped frames** — a scroll that stutters, an animation that hitches. A frame
has about 16.7 ms at 60 Hz and 8.3 ms at 120 Hz, split between the UI thread (build, layout, paint recording) and the raster
thread (turning layers into pixels). This lesson covers how to *measure* first, then the handful of fixes that solve most
real problems, each with numbers from tests rather than folklore. One of the experiments contradicts a piece of common advice.

## Rule 1: measure in profile mode, on a device

```bash
flutter run --profile
```

- **Debug mode** is JIT-compiled with assertions and extra checks; it can be many times slower and is useless for judging
  performance.
- **Profile mode** is AOT-compiled like release, but keeps enough instrumentation for DevTools.
- **Emulators and simulators** don't have your users' GPUs or thermals. Use a real, preferably mid-range, device.

Then open **Flutter DevTools** (the URL is printed by `flutter run`, or use your IDE):

- **Performance view**: a frame chart where each bar is a frame, split into UI and raster time; slow frames are highlighted.
  Click one to see a timeline of what ran — which `build` methods, layouts, paints.
- **"Track widget builds"** shows which widgets rebuilt in a frame and how often.
- **"Highlight repaints"** / the performance overlay (`showPerformanceOverlay: true` on `MaterialApp`) show what repaints
  and the two thread graphs live on the device.
- **CPU profiler** for Dart hot spots; **Memory view** for leaks and allocation spikes.

!!! note "What this course could run"
    Profile mode needs a device plus a working mobile or desktop toolchain; this course's machine had no device attached
    and couldn't build for Android, iOS or macOS (see
    [Level 3 · 09](../level-3/09-integration-tests.md)), so the DevTools workflow above is described, not demonstrated.
    The experiments below measure *work done* — build counts and widget-test timings — which is deterministic and
    reproducible anywhere, and is what usually explains the slow frames DevTools shows.

## The experiments

```dart title="perf_demo.dart"
import 'package:flutter/material.dart';

final builds = <String, int>{};
void count(String name) => builds[name] = (builds[name] ?? 0) + 1;

class Row_ extends StatelessWidget {
  const Row_(this.i, {super.key});
  final int i;
  @override
  Widget build(BuildContext context) {
    count('row');
    return SizedBox(height: 48, child: Text('Item $i'));
  }
}

/// 1. Eager vs lazy lists.
Widget eagerList(int n) => ListView(children: [for (var i = 0; i < n; i++) Row_(i)]);
Widget lazyList(int n) => ListView.builder(itemCount: n, itemExtent: 48, itemBuilder: (_, i) => Row_(i));

/// 2. Where setState lives decides how much rebuilds.
class Expensive extends StatelessWidget {
  const Expensive({super.key});
  @override
  Widget build(BuildContext context) {
    count('expensive');
    return const Text('a big, static subtree');
  }
}

class CounterPageBad extends StatefulWidget {
  const CounterPageBad({super.key});
  @override
  State<CounterPageBad> createState() => _CounterPageBadState();
}

class _CounterPageBadState extends State<CounterPageBad> {
  int n = 0;
  @override
  Widget build(BuildContext context) {
    count('page');
    return Column(children: [
      // ignore: prefer_const_constructors
      Expensive(), // not const: a new instance every build, so it rebuilds every time
      Text('$n'),
      TextButton(onPressed: () => setState(() => n++), child: const Text('+')),
    ]);
  }
}

class CounterPageGood extends StatelessWidget {
  const CounterPageGood({super.key});
  @override
  Widget build(BuildContext context) {
    count('page');
    return const Column(children: [
      Expensive(), // const: identical instance, skipped on rebuild
      _Counter(), // the state lives as low as possible
    ]);
  }
}

class _Counter extends StatefulWidget {
  const _Counter();
  @override
  State<_Counter> createState() => _CounterState();
}

class _CounterState extends State<_Counter> {
  int n = 0;
  @override
  Widget build(BuildContext context) {
    count('counter');
    return Column(children: [Text('$n'), TextButton(onPressed: () => setState(() => n++), child: const Text('+'))]);
  }
}

/// 3. Fading: rebuild-per-frame vs render-object-level animation.
class FadeWithSetState extends StatefulWidget {
  const FadeWithSetState({super.key});
  @override
  State<FadeWithSetState> createState() => _FadeWithSetStateState();
}

class _FadeWithSetStateState extends State<FadeWithSetState> with SingleTickerProviderStateMixin {
  late final c = AnimationController(vsync: this, duration: const Duration(seconds: 1))
    ..addListener(() => setState(() {}))
    ..forward();
  @override
  void dispose() {
    c.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    count('fade-setState');
    return Opacity(opacity: c.value, child: const Expensive());
  }
}

class FadeWithTransition extends StatefulWidget {
  const FadeWithTransition({super.key});
  @override
  State<FadeWithTransition> createState() => _FadeWithTransitionState();
}

class _FadeWithTransitionState extends State<FadeWithTransition> with SingleTickerProviderStateMixin {
  late final c = AnimationController(vsync: this, duration: const Duration(seconds: 1))..forward();
  @override
  void dispose() {
    c.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    count('fade-transition');
    return FadeTransition(opacity: c, child: const Expensive());
  }
}
```

```dart title="perf_test.dart"
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:l2/m/perf_demo.dart';

void main() {
  setUp(builds.clear);

  testWidgets('1. eager vs lazy list of 100,000', (tester) async {
    const n = 100000;
    Future<int> time(Widget Function() make) async {
      await tester.pumpWidget(const SizedBox());
      final sw = Stopwatch()..start();
      await tester.pumpWidget(MaterialApp(home: make()));
      return sw.elapsedMilliseconds;
    }

    // Warm up both paths once so JIT compilation doesn't distort the comparison.
    await time(() => eagerList(n));
    await time(() => lazyList(n));

    for (var round = 1; round <= 3; round++) {
      builds.clear();
      final eager = await time(() => eagerList(n));
      final eagerBuilt = builds['row'];
      builds.clear();
      final lazy = await time(() => lazyList(n));
      print('round $round: ListView(children) $eager ms, built $eagerBuilt rows | '
          'ListView.builder $lazy ms, built ${builds['row']} rows');
    }
    builds.clear();
    await tester.drag(find.byType(ListView), const Offset(0, -2000));
    await tester.pump();
    print('builder, after scrolling 2000px: built ${builds['row']} more rows');
  });

  testWidgets('2. where setState lives', (tester) async {
    await tester.pumpWidget(const MaterialApp(home: CounterPageBad()));
    builds.clear();
    for (var i = 0; i < 5; i++) {
      await tester.tap(find.text('+'));
      await tester.pump();
    }
    print('bad  (5 taps): $builds');
    await tester.pumpWidget(const MaterialApp(home: CounterPageGood()));
    builds.clear();
    for (var i = 0; i < 5; i++) {
      await tester.tap(find.text('+'));
      await tester.pump();
    }
    print('good (5 taps): $builds');
  });

  testWidgets('3. animating opacity for one second at 60 fps', (tester) async {
    await tester.pumpWidget(const MaterialApp(home: FadeWithSetState()));
    for (var i = 0; i < 60; i++) {
      await tester.pump(const Duration(milliseconds: 16));
    }
    print('setState + Opacity: $builds');
    builds.clear();
    await tester.pumpWidget(const MaterialApp(home: FadeWithTransition()));
    for (var i = 0; i < 60; i++) {
      await tester.pump(const Duration(milliseconds: 16));
    }
    print('FadeTransition:     $builds');
  });
}
```

```console
$ flutter test test/m/perf_test.dart
round 1: ListView(children) 14 ms, built 18 rows | ListView.builder 11 ms, built 18 rows
round 2: ListView(children) 15 ms, built 18 rows | ListView.builder 11 ms, built 18 rows
round 3: ListView(children) 16 ms, built 18 rows | ListView.builder 9 ms, built 18 rows
builder, after scrolling 2000px: built 24 more rows
bad  (5 taps): {page: 5, expensive: 5}
good (5 taps): {counter: 5}
setState + Opacity: {fade-setState: 61, expensive: 1}
FadeTransition:     {fade-transition: 1, expensive: 1}
00:00 +3: All tests passed!
```

### 1. Lists: the myth and the real cost

A common claim is that `ListView(children: [...])` builds every item. The counts say otherwise (as
[Level 1 · 07](../level-1/07-lists-keys.md) first showed): with 100,000 items, **both**
forms built only the 18 rows visible on the 800×600 test screen (scrolling 2000 px then built 24 more). Both use a lazy
sliver underneath, so elements, layout and paint happen only for visible children.

The real difference is that `ListView(children:)` requires all 100,000 *widget objects* to exist up front — the
`for` loop allocates every one, every time that list is rebuilt. In this test that cost 4–7 ms per build over the builder
version (measured after a warm-up round, because in an earlier version of this test without one, whichever list was built
first paid for one-time compilation and looked an order of magnitude slower).
Cheap here, because `Row_` is tiny. If constructing each item means formatting dates, parsing, or building nested widget
trees, that up-front cost is multiplied by the item count. So the advice stands, for a more precise reason: **use
`ListView.builder` (or `SliverList.builder`) for long or data-driven lists**, and give it an `itemExtent` or
`prototypeItem` when rows have a fixed height, so the scroll view can compute positions without laying out rows to measure
them.

### 2. Where `setState` lives

In the "bad" page the counter's state is in the page, so every tap rebuilt the page **and** `Expensive` (5 and 5) — because
`Expensive()` wasn't `const`, each page build created a new instance, and a different instance at the same position must be
rebuilt. The "good" version moved the state into a small `_Counter` and made the static parts `const`: five taps, five
rebuilds of the counter, nothing else. Two independent fixes, both worth applying:

- **Push state down** to the smallest widget that needs it (or use selective subscriptions: `select`, `Consumer`,
  `BlocBuilder` with `buildWhen`).
- **Use `const`** for static subtrees. The `prefer_const_constructors` lint finds them for you (`flutter_lints` enables it).

### 3. Animations without rebuilds

Driving opacity with `setState` in a controller listener rebuilt the widget **61 times** in one second (once per frame,
plus the first). `FadeTransition` built it **once**; the animation updated the render object's opacity directly each frame.
`Expensive` was built only once in both cases because it was a `const` child — without `const` (or the `child` parameter
of `AnimatedBuilder`), it too would have rebuilt 61 times. Also note: `Opacity` with values between 0 and 1 can require an
offscreen layer (`saveLayer`) for some content, which is raster-thread work; `FadeTransition` and `AnimatedOpacity` are the
efficient choices.

## The usual suspects, in rough order of frequency

1. **Too much rebuilding** — state too high, missing `const`, `watch` where `select` would do.
2. **Expensive work in `build`** — sorting, filtering, parsing, `DateFormat(...)` construction. Compute once and cache, or
   move it into your state layer.
3. **Non-lazy lists and grids** for large data sets; `shrinkWrap: true` on long lists inside other scrollables (it forces
   laying out every item to measure the total height).
4. **Intrinsic layout** (`IntrinsicHeight`, some `Table` configurations) in repeated rows.
5. **Raster-thread costs**: `saveLayer` (from `Opacity`, `ShaderMask`, `ColorFilter`, some clips), large blurs
   (`BackdropFilter`), huge images decoded at full resolution — use `cacheWidth`/`cacheHeight` on `Image` to decode at display
   size.
6. **CPU work on the UI isolate** — move it to an isolate ([Level 3 · 06](../level-3/06-isolates.md)).
7. **First-run shader compilation jank** on some backends: the Impeller renderer (default on iOS and on modern Android) was
   designed to precompile its shaders and largely removes this class of jank; where the older Skia backend is still in use,
   jank on first appearance of an effect can still occur.

## Regression protection

Performance fixes rot. Keep them with tests like the ones above — asserting build counts (`expect(builds['expensive'],
1)`) is crude but catches "someone removed the `const`" regressions in CI. For frame timing on real devices, an integration
test can wrap a scroll in `traceAction` and fail if the 90th-percentile frame build time crosses a budget.

## How It Actually Works

Every one of these fixes reduces one of the dirty sets from [lesson 01](01-rendering-pipeline.md). `setState` marks one
element dirty, but its `build` produces new widgets for its whole subtree, and every child whose widget isn't *identical* to
last frame's (and that isn't a `const` instance) must be updated — which for `StatelessWidget`s means running `build` again,
recursively. `const` canonicalizes instances at compile time, so `identical(old, new)` is true and `updateChild` returns
immediately. `FadeTransition` skips the build phase entirely: the `RenderAnimatedOpacity` listens to the animation and only
calls `markNeedsPaint`, and because opacity is applied as an `OpacityLayer` at composite time, even the child's painting can
be reused. Lazy slivers ask their delegate for children only as layout reaches each index, and `itemExtent` lets
`RenderSliverFixedExtentList` compute which indices are visible with arithmetic instead of laying children out.

## Common mistakes

- **Optimizing in debug mode.**
- **Guessing** instead of opening the performance view.
- **Adding `RepaintBoundary` everywhere** — each one costs memory and compositing.
- **Premature `compute`** for tiny work, or `const` mania that obscures code without measurable gain.
- **Ignoring the raster thread** when the UI thread looks fine.

## Exercise

1. Add `print` counters to the Level 2 reading-list app's widgets and tick a rating. What rebuilt that didn't need to? Fix it
   and add a regression assertion.
2. Make `Row_` expensive (format a `DateTime` and build a nested `Row` with an icon) and re-run experiment 1. How does the gap
   between the two list types change?
3. Replace `FadeTransition` with `AnimatedBuilder` + `Opacity` *with* the `child` parameter. Count builds of the builder and of
   `Expensive`.
4. On a real device, run the app with `--profile`, open DevTools, scroll a long list, and find the slowest frame. What was in
   it?
