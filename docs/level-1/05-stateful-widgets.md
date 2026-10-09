---
description: "StatefulWidget and State explained by logging every lifecycle call: initState, didUpdateWidget, dispose, coalesced setState, and the timer leak bug."
---

# 05 · StatefulWidget, setState & the State Lifecycle

A `StatelessWidget` can only describe what it's given. Anything that changes on its own — a
counter, a text field's contents, a countdown, an animation — needs **state**: data that lives
longer than one `build` call. Flutter splits a stateful widget into two classes: the immutable
`StatefulWidget` (configuration, recreated on every parent rebuild) and a long-lived `State`
object (the mutable data, kept across rebuilds). This lesson logs every lifecycle method of a
real `State` under a widget test, so you can see exactly when each runs — and what goes wrong
when cleanup is forgotten.

## A countdown that logs its lifecycle

```dart title="lifecycle.dart"
import 'dart:async';
import 'package:flutter/material.dart';

final lifecycleLog = <String>[];

/// A countdown that ticks once per second and reports each lifecycle call.
class Countdown extends StatefulWidget {
  const Countdown({super.key, required this.from, this.onDone});
  final int from;
  final VoidCallback? onDone;

  @override
  State<Countdown> createState() {
    lifecycleLog.add('createState');
    return _CountdownState();
  }
}

class _CountdownState extends State<Countdown> {
  late int _remaining;
  Timer? _timer;

  @override
  void initState() {
    super.initState();
    lifecycleLog.add('initState(from=${widget.from})');
    _remaining = widget.from;
    _timer = Timer.periodic(const Duration(seconds: 1), (_) => _tick());
  }

  void _tick() {
    setState(() => _remaining--);
    if (_remaining == 0) {
      _timer?.cancel();
      widget.onDone?.call();
    }
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    lifecycleLog.add('didChangeDependencies');
  }

  @override
  void didUpdateWidget(Countdown oldWidget) {
    super.didUpdateWidget(oldWidget);
    lifecycleLog.add('didUpdateWidget(${oldWidget.from} -> ${widget.from})');
    if (oldWidget.from != widget.from) {
      _remaining = widget.from; // parent asked for a new countdown
    }
  }

  @override
  void dispose() {
    lifecycleLog.add('dispose');
    _timer?.cancel(); // without this, the timer keeps calling setState on a dead State
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    lifecycleLog.add('build($_remaining)');
    return Text('$_remaining', style: Theme.of(context).textTheme.displaySmall);
  }
}
```

The structure to memorise:

- `Countdown` (the widget) holds configuration: `from` and `onDone`. It's `const`-constructible and
  immutable.
- `createState()` creates the `_CountdownState` — once per *place in the tree*, not once per build.
  (Logging inside `createState` is only for this demonstration; `flutter analyze` flags it with
  `no_logic_in_create_state`, and real code should do nothing there but return the state.)
- `_CountdownState` holds `_remaining` and the `Timer`. It reads its widget's configuration through
  the `widget` getter.
- `setState(fn)` runs `fn` to change fields and **schedules a rebuild**. It doesn't rebuild
  immediately.

## The lifecycle, observed

```dart title="lifecycle_test.dart"
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hello/l1/lifecycle.dart';

Widget app(Widget child) => MaterialApp(home: Scaffold(body: Center(child: child)));

void main() {
  setUp(lifecycleLog.clear);

  testWidgets('lifecycle of a State', (tester) async {
    var done = false;
    await tester.pumpWidget(app(Countdown(from: 3, onDone: () => done = true)));
    print('mount:          $lifecycleLog'); lifecycleLog.clear();

    await tester.pump(const Duration(seconds: 1));
    print('after 1s:       $lifecycleLog  text=${(tester.widget<Text>(find.byType(Text))).data}');
    lifecycleLog.clear();

    // Parent rebuilds with new configuration: same State, didUpdateWidget.
    await tester.pumpWidget(app(Countdown(from: 5, onDone: () => done = true)));
    print('new config:     $lifecycleLog'); lifecycleLog.clear();

    await tester.pump(const Duration(seconds: 5));
    print('after 5s more:  $lifecycleLog  done=$done'); lifecycleLog.clear();

    // Removing the widget disposes the State.
    await tester.pumpWidget(app(const SizedBox()));
    print('removed:        $lifecycleLog');
  });

  testWidgets('different type at the same spot = new State', (tester) async {
    await tester.pumpWidget(app(const Countdown(from: 3)));
    lifecycleLog.clear();
    await tester.pumpWidget(app(const Padding(padding: EdgeInsets.zero, child: Countdown(from: 3))));
    print('wrapped in Padding: $lifecycleLog');
    await tester.pumpWidget(app(const SizedBox()));
  });
}
```

```console
$ flutter test test/l1/lifecycle_test.dart
mount:          [createState, initState(from=3), didChangeDependencies, build(3)]
after 1s:       [build(2)]  text=2
new config:     [didUpdateWidget(3 -> 5), build(5)]
after 5s more:  [build(0)]  done=true
removed:        [dispose]
wrapped in Padding: [createState, initState(from=3), didChangeDependencies, build(3), dispose]
00:00 +2: All tests passed!
```

(`tester.pump(duration)` advances the test's fake clock, so timers fire instantly — a five-second
countdown takes milliseconds to test.)

| Call | When | Use it for |
|---|---|---|
| `createState` | widget inserted at a new place | creating the `State` (nothing else) |
| `initState` | once, right after creation | one-time setup: controllers, timers, subscriptions |
| `didChangeDependencies` | after `initState`, and whenever an inherited widget it depends on changes | work that needs `Theme.of`, `MediaQuery.of`, etc. |
| `build` | every time the state is dirty or its parent rebuilds | describing UI — nothing else |
| `didUpdateWidget` | parent rebuilt with a new widget of the same type | reacting to changed configuration |
| `dispose` | permanently removed | cancel timers, close streams, dispose controllers |

Four observations from the run:

1. **New configuration keeps the same `State`.** When the parent pumped `Countdown(from: 5)`,
   Flutter didn't create a new state; it called `didUpdateWidget(3 -> 5)`. If the state had ignored
   that, the countdown would have kept counting from 2. Whenever a field derived from `widget` should
   follow changes, handle `didUpdateWidget`.
2. **Several `setState` calls, one build.** Advancing five seconds fired the timer five times, each
   calling `setState`, but only one `build(0)` was logged. `setState` marks the state dirty; the
   rebuild happens once, at the next frame. You never need to "batch" `setState` calls yourself.
3. **`dispose` on removal.** Replacing the countdown with a `SizedBox` disposed its state.
4. **Changing the tree's shape loses state.** Wrapping the same `Countdown` in a `Padding` created a
   *new* state (`createState`, `initState` …) and disposed the old one. At that position the old
   element was a `Countdown`, the new widget is a `Padding` — different types — so the subtree was
   rebuilt from scratch. Conditional wrappers (`showBorder ? Padding(child: x) : x`) silently reset
   state for exactly this reason; [lesson 07](07-lists-keys.md) shows how keys interact with it.

## The bug every Flutter developer writes once

Comment out `_timer?.cancel()` in `dispose`, mount the countdown, remove it, and let one second pass:

```text
══╡ EXCEPTION CAUGHT BY FLUTTER TEST FRAMEWORK ╞════════════════════════════════════════════════════
The following assertion was thrown running a test:
setState() called after dispose(): _CountdownState#db9a2(lifecycle state: defunct, not mounted)
This error happens if you call setState() on a State object for a widget that no longer appears in
the widget tree (e.g., whose parent widget no longer includes the widget in its build). This error
can occur when code calls setState() from a timer or an animation callback.
The preferred solution is to cancel the timer or stop listening to the animation in the dispose()
callback. Another solution is to check the "mounted" property of this object before calling
setState() to ensure the object is still in the tree.
...
```

The timer held a reference to the dead `State` and kept calling `_tick`. The message is unusually
helpful: cancel in `dispose`. Checking `if (!mounted) return;` before `setState` is the right fix for
a *one-off* async callback (an awaited network call that finishes after the user navigated away),
but for repeating sources — timers, stream subscriptions, animation controllers — cancel them in
`dispose`, or they leak.

## Where should state live?

`setState` is perfect for state that belongs to one widget and nothing else cares about: whether a
panel is expanded, the current text in a search box, a countdown's value. When several widgets need
the same data, or it must survive navigation, it moves up the tree or into a state-management
solution — [Level 2](../level-2/index.md) covers lifting state, `InheritedWidget`, Provider,
Riverpod and Bloc. A useful rule: start with `setState`, and move state out when a second widget needs
it.

## How It Actually Works

A `StatefulWidget` creates a `StatefulElement`. When the element is mounted, it calls
`widget.createState()`, sets `state._widget` and `state._element`, and calls `initState` then
`didChangeDependencies`, then builds. The element keeps the same `State` for as long as the element
lives. When a parent rebuilds and `Widget.canUpdate(old, new)` holds (same type and key), the element
stores the new widget, calls `state.didUpdateWidget(oldWidget)`, and rebuilds.

`setState(fn)` calls `fn` synchronously, then `_element.markNeedsBuild()`, which adds the element to the
owner's dirty list and asks the engine for a frame if one isn't already scheduled. During the next
frame's build phase, each dirty element rebuilds once — hence the coalescing. When an element is
removed, it's first **deactivated** (it could still be re-inserted elsewhere in the same frame, e.g.
with a `GlobalKey`); if it isn't reused by the end of the frame, it's **unmounted** and `dispose` runs.
After that, `mounted` is false and `setState` asserts, as seen above.

## Common mistakes

- **Not cancelling timers, subscriptions or controllers in `dispose`.**
- **Initialising state from `widget` in `initState` and never updating it** in `didUpdateWidget`.
- **Calling `Theme.of(context)` or `MediaQuery.of(context)` in `initState`** — use
  `didChangeDependencies` or `build`.
- **Doing async work in `setState`'s callback.** The callback must be synchronous; await first, then
  call `setState` with the result (checking `mounted`).
- **Conditionally wrapping a stateful widget**, losing its state whenever the condition flips.

## Exercise

1. Add a "Pause" button that stops and restarts the timer. Where do you store whether it's paused?
2. Remove `didUpdateWidget` from `_CountdownState` and change the test to expect the bug: what number
   is shown after the parent switches `from` to 5?
3. Make the `Padding` wrapper conditional on a `bool` and write a test that proves toggling it resets
   the countdown. Then fix it by always including the `Padding` (with `EdgeInsets.zero` when off).
4. Write a widget that awaits a fake two-second "network" `Future` in `initState` and shows the result.
   Write a test that removes it after one second and confirm no `setState` error occurs thanks to a
   `mounted` check.
