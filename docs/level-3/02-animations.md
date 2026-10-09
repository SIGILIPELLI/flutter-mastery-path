---
description: "Flutter animations measured frame by frame: AnimatedContainer, AnimationController with Tween and curves, staggered intervals, leak checks."
---

# 02 · Animations: Implicit & Explicit

Flutter animations come in two flavours. **Implicit** animations: you change a property and the widget animates from
the old value to the new one by itself. **Explicit** animations: you own an `AnimationController` and decide when it
runs, repeats, reverses or stops. Start with implicit; reach for explicit when you need control (looping, sequencing,
reacting to gestures). This lesson samples every animation at fixed points in fake time, so you can see the numbers
instead of trusting a GIF.

## Implicit: change a value, get an animation

```dart title="anim_demo.dart (implicit)"
import 'package:flutter/material.dart';

/// Implicit: just change a value; the widget animates from old to new.
class ExpandingCard extends StatefulWidget {
  const ExpandingCard({super.key});
  @override
  State<ExpandingCard> createState() => _ExpandingCardState();
}

class _ExpandingCardState extends State<ExpandingCard> {
  bool open = false;
  @override
  Widget build(BuildContext context) => GestureDetector(
        onTap: () => setState(() => open = !open),
        child: AnimatedContainer(
          key: const Key('card'),
          duration: const Duration(milliseconds: 400),
          curve: Curves.easeInOut,
          width: 200,
          height: open ? 300 : 100,
          color: open ? Colors.indigo : Colors.grey,
        ),
      );
}
```

`AnimatedContainer` remembers its previous `height` and `color` and tweens to the new ones over `duration` with `curve`.
Its siblings work the same way: `AnimatedOpacity`, `AnimatedAlign`, `AnimatedPadding`, `AnimatedDefaultTextStyle`,
`AnimatedSwitcher` (cross-fades between two different children), and `TweenAnimationBuilder` for animating any value of
your own to a target.

## Explicit: own the controller

```dart title="anim_demo.dart (explicit)"
/// Explicit: we own an AnimationController and drive it.
class PulsingDot extends StatefulWidget {
  const PulsingDot({super.key});
  @override
  State<PulsingDot> createState() => PulsingDotState();
}

class PulsingDotState extends State<PulsingDot> with SingleTickerProviderStateMixin {
  late final AnimationController controller = AnimationController(
    vsync: this, // ties ticks to the screen refresh; pauses when off-screen
    duration: const Duration(seconds: 1),
  );
  late final Animation<double> scale =
      Tween<double>(begin: 1, end: 1.5).animate(CurvedAnimation(parent: controller, curve: Curves.easeOut));

  @override
  void initState() {
    super.initState();
    controller.repeat(reverse: true);
  }

  @override
  void dispose() {
    controller.dispose(); // forgetting this leaks a Ticker
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => AnimatedBuilder(
        animation: scale,
        // child is built once and reused on every tick
        child: Container(width: 20, height: 20, decoration: const BoxDecoration(color: Colors.red, shape: BoxShape.circle)),
        builder: (context, child) => Transform.scale(scale: scale.value, child: child),
      );
}
```

The moving parts:

- **`AnimationController`** produces a value from 0.0 to 1.0 over `duration`, one step per frame. You call `forward`,
  `reverse`, `repeat`, `stop`, `animateTo`.
- **`vsync: this`** with `SingleTickerProviderStateMixin` gives it a `Ticker` — a per-frame callback that's muted when
  the widget's route isn't visible, so off-screen animations don't burn battery. (Use `TickerProviderStateMixin` for
  several controllers.)
- **`CurvedAnimation`** remaps 0→1 through an easing curve; **`Tween`** maps 0→1 onto the range you want (1.0→1.5 here).
- **`AnimatedBuilder`** rebuilds only its `builder` on each tick. The `child` (the red dot) is built once and passed in —
  an important optimization for anything non-trivial.
- **`dispose` the controller.** Always.

Built-in `*Transition` widgets (`FadeTransition`, `SlideTransition`, `ScaleTransition`, `RotationTransition`) take an
`Animation` directly and are even cheaper than an `AnimatedBuilder`, because they skip the build phase and update
the render object's property on each tick.

## Staggered: one controller, several intervals

```dart title="anim_demo.dart (staggered)"
/// Staggered: one controller, several intervals.
class StaggeredIntro extends StatefulWidget {
  const StaggeredIntro({super.key});
  @override
  State<StaggeredIntro> createState() => StaggeredIntroState();
}

class StaggeredIntroState extends State<StaggeredIntro> with SingleTickerProviderStateMixin {
  late final AnimationController c = AnimationController(vsync: this, duration: const Duration(milliseconds: 1000))..forward();
  late final Animation<double> titleOpacity = CurvedAnimation(parent: c, curve: const Interval(0.0, 0.4));
  late final Animation<Offset> bodySlide =
      Tween(begin: const Offset(0, 1), end: Offset.zero).animate(CurvedAnimation(parent: c, curve: const Interval(0.3, 0.8, curve: Curves.easeOut)));
  late final Animation<double> buttonOpacity = CurvedAnimation(parent: c, curve: const Interval(0.7, 1.0));

  @override
  void dispose() {
    c.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => Column(children: [
        FadeTransition(opacity: titleOpacity, child: const Text('Welcome')),
        SlideTransition(position: bodySlide, child: const Text('Here is how it works')),
        FadeTransition(opacity: buttonOpacity, child: const Text('Get started')),
      ]);
}
```

`Interval(0.3, 0.8)` means "stay at 0 until the controller reaches 0.3, go to 1 by 0.8, then stay at 1". Overlapping
intervals on one controller give you a choreographed sequence that's always in sync and can be reversed as a whole.

## Sampling the animations

```dart title="anim_test.dart"
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:l2/a/anim_demo.dart';

void main() {
  testWidgets('implicit: AnimatedContainer height over time', (tester) async {
    await tester.pumpWidget(const MaterialApp(home: Center(child: ExpandingCard())));
    await tester.tap(find.byKey(const Key('card')));
    final heights = <String>[];
    await tester.pump(); // start
    for (var ms = 0; ms <= 400; ms += 100) {
      heights.add('${ms}ms:${tester.getSize(find.byKey(const Key('card'))).height.toStringAsFixed(1)}');
      await tester.pump(const Duration(milliseconds: 100));
    }
    print(heights.join('  '));
  });

  testWidgets('explicit: repeating controller', (tester) async {
    await tester.pumpWidget(const MaterialApp(home: Center(child: PulsingDot())));
    final state = tester.state<PulsingDotState>(find.byType(PulsingDot));
    final samples = <String>[];
    for (var i = 0; i <= 8; i++) {
      samples.add('${i * 250}ms:${state.scale.value.toStringAsFixed(3)}(${state.controller.status.name})');
      await tester.pump(const Duration(milliseconds: 250));
    }
    print(samples.join('\n'));
    // pumpAndSettle would time out: a repeating animation never settles.
    await tester.pumpWidget(const SizedBox()); // dispose it
  });

  testWidgets('staggered intervals', (tester) async {
    await tester.pumpWidget(const MaterialApp(home: StaggeredIntro()));
    final s = tester.state<StaggeredIntroState>(find.byType(StaggeredIntro));
    for (final ms in [0, 200, 400, 600, 800, 1000]) {
      if (ms > 0) await tester.pump(const Duration(milliseconds: 200));
      print('${ms.toString().padLeft(4)}ms title=${s.titleOpacity.value.toStringAsFixed(2)} '
          'bodyY=${s.bodySlide.value.dy.toStringAsFixed(2)} button=${s.buttonOpacity.value.toStringAsFixed(2)}');
    }
  });

}
```

```console
$ flutter test test/a/anim_test.dart
0ms:100.0  100ms:125.7  200ms:200.0  300ms:274.3  400ms:300.0
0ms:1.000(forward)
250ms:1.189(forward)
500ms:1.342(forward)
750ms:1.453(forward)
1000ms:1.500(reverse)
1250ms:1.453(reverse)
1500ms:1.342(reverse)
1750ms:1.189(reverse)
2000ms:1.000(forward)
   0ms title=0.00 bodyY=1.00 button=0.00
 200ms title=0.50 bodyY=1.00 button=0.00
 400ms title=1.00 bodyY=0.69 button=0.00
 600ms title=1.00 bodyY=0.21 button=0.00
 800ms title=1.00 bodyY=0.00 button=0.33
1000ms title=1.00 bodyY=0.00 button=1.00
00:00 +3: All tests passed!
```

Read the numbers:

- **`easeInOut`** is symmetric: 25.7 px of the 200 px change in the first quarter, the steep middle (to 200 at half-time),
  and the same 25.7 px in the last quarter. A `linear` curve would give 150/200/250.
- **`easeOut`** front-loads motion: the dot covered 0.189 of its 0.5 growth (38%) in the first quarter-second and only
  0.047 in the last. With `repeat(reverse: true)` the status flips to `reverse` at the top and back to `forward` at the
  bottom, and the values retrace the same curve.
- **Staggering:** the title faded in over 0–400 ms (interval 0.0–0.4, linear, so 0.5 at 200 ms); the body didn't move until
  300 ms and arrived by 800 ms; the button waited until 700 ms (0.33 at 800 ms is a third of the way through 0.7–1.0).

Note that the repeating test never calls `pumpAndSettle` — a repeating animation never settles, so it would time out.

## Forgetting to dispose

What happens when a `State` creates a controller and never disposes it? A separate test, which **fails on purpose**:

```dart title="leak_test.dart"
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

// This test FAILS on purpose, to show the diagnostics for a leaked AnimationController.
void main() {
  testWidgets('forgetting dispose is reported', (tester) async {
    await tester.pumpWidget(const MaterialApp(home: LeakyDot()));
    await tester.pumpWidget(const SizedBox());
    final e = tester.takeException();
    print(e.toString().split('\n').take(2).join(' | '));
  });
}

class LeakyDot extends StatefulWidget {
  const LeakyDot({super.key});
  @override
  State<LeakyDot> createState() => _LeakyDotState();
}

class _LeakyDotState extends State<LeakyDot> with SingleTickerProviderStateMixin {
  late final AnimationController c;
  @override
  void initState() {
    super.initState();
    c = AnimationController(vsync: this, duration: const Duration(seconds: 1))..repeat();
  }

  @override
  Widget build(BuildContext context) => const SizedBox();
  // no dispose()
}
```

```console
$ flutter test test/a/leak_test.dart
_LeakyDotState#1be73(ticker active) was disposed with an active Ticker. | _LeakyDotState created a Ticker via its SingleTickerProviderStateMixin, but at the time dispose() was called on the mixin, that Ticker was still active. The Ticker must be disposed before calling super.dispose().
══╡ EXCEPTION CAUGHT BY SCHEDULER LIBRARY ╞═════════════════════════════════════════════════════════
The following message was thrown:
An animation is still running even after the widget tree was disposed.
...
00:00 +0 -1: Some tests failed.
```

Two separate safety nets fired: the ticker mixin's debug assertion when the `State` was disposed, and the test framework's
check for frame callbacks still scheduled at the end of the test. In a running app, the leak keeps scheduling frames for a
widget that no longer exists.

(While writing this test, the first version declared the controller as `late final c = AnimationController(...)..repeat();`
and *passed* — because a `late` field is only initialized when first read, and nothing read it. No controller, no leak.
The explicit-animation demo above reads its controller in `initState`, which is what makes it start.)

## Choosing

| Need | Use |
|---|---|
| A property should animate when state changes | `AnimatedFoo` / `TweenAnimationBuilder` |
| Swap one widget for another with a transition | `AnimatedSwitcher` |
| Shared element between routes | `Hero` |
| Loop, sequence, scrub, or drive from a gesture | `AnimationController` + `*Transition` / `AnimatedBuilder` |
| Physics (fling, spring) | `controller.animateWith(SpringSimulation(...))` |
| Designer-made vector animations | packages such as `rive` or `lottie` |

## How It Actually Works

Each frame, the `SchedulerBinding` runs *transient callbacks* before building. Every active `Ticker` registered one; it
calls the controller with the elapsed time since start. The controller converts elapsed time into its 0–1 value (or runs
a `Simulation` for physics), then notifies listeners. `CurvedAnimation` and `Tween.animate` don't store values — they're
views that compute `curve.transform(parent.value)` and `lerp(begin, end, t)` on demand.

Listeners decide how much work happens. `AnimatedBuilder` calls `setState` internally, so its builder runs in the build
phase. `FadeTransition` instead passes the animation to `RenderAnimatedOpacity`, which listens and calls
`markNeedsPaint` — skipping build *and* layout. That's why transitions are the most efficient option for opacity and
transforms. Implicit widgets are just `State`s that create a controller for you and restart it from the current value
whenever `didUpdateWidget` sees a new target.

## Common mistakes

- **Not disposing controllers**, shown above.
- **Creating the controller in `build`.** It restarts on every rebuild.
- **`setState` in a controller listener** to rebuild a whole screen on every frame. Use `AnimatedBuilder` scoped to the
  part that moves, or a `*Transition`.
- **Animating layout properties** (width/height) of large subtrees when a `Transform` would do — layout every frame is far
  more expensive than painting with a transform.
- **Ignoring reduced motion.** Check `MediaQuery.disableAnimationsOf(context)` and shorten or skip non-essential
  animations ([lesson 08](08-a11y-i18n.md)).

## Exercise

1. Change `ExpandingCard`'s curve to `Curves.linear`, then to `Curves.elasticOut`, and record the sampled heights. Which
   one overshoots 300?
2. Make the pulsing dot pause when tapped and resume on the next tap, from the value where it stopped.
3. Reverse the staggered intro when a "Back" button is pressed. What order do the elements leave in?
4. Replace `AnimatedBuilder` + `Transform.scale` with `ScaleTransition`. Use a `buildLog` list (as in Level 2 · 03) to
   prove the builder no longer runs on each tick.
