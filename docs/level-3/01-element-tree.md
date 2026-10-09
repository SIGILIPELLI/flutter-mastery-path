---
description: "How Flutter's widget, element and render trees relate: when State is kept or lost, why keys matter when reordering, and what BuildContext is."
---

# 01 · Widgets, Elements & BuildContext

Levels 1 and 2 used a simplified model: widgets describe UI, `build` returns more widgets, `setState` rebuilds. That
model explains most code, but not a set of puzzling bugs: a text field that loses its contents when you wrap it in a
`Padding`, checkboxes whose ticks stay put when the list is re-sorted, "Looking up a deactivated widget's ancestor is
unsafe". All of these come from the layer underneath widgets: **elements**. This lesson makes that layer visible with
five small experiments.

## Three trees

| Tree | Objects | Lifetime | Job |
|---|---|---|---|
| Widget | `Text`, `Padding`, your widgets | rebuilt constantly, immutable, cheap | *configuration* |
| Element | `StatelessElement`, `StatefulElement`, … | long-lived, mutable | *identity and position*; holds `State` |
| Render | `RenderParagraph`, `RenderPadding`, … | long-lived, created by some elements | layout, paint, hit-testing |

Each frame you hand Flutter a fresh widget tree. Flutter walks it alongside the existing element tree and, at each
position, decides whether the existing element can be **updated** with the new widget or must be **replaced**. The rule
is `Widget.canUpdate`: same `runtimeType` **and** same `key`. Everything in this lesson follows from that one rule.

## The experiments

A stateful tile that counts taps and logs its lifecycle:

```dart title="element_demo.dart"
import 'package:flutter/material.dart';

/// A tile that remembers how many times it was tapped (state lives in State, i.e. in the element).
class Tally extends StatefulWidget {
  const Tally({super.key, required this.label});
  final String label;
  @override
  State<Tally> createState() => _TallyState();
}

final lifecycle = <String>[];

class _TallyState extends State<Tally> {
  int taps = 0;
  @override
  void initState() {
    super.initState();
    lifecycle.add('initState ${widget.label}');
  }

  @override
  void didUpdateWidget(Tally old) {
    super.didUpdateWidget(old);
    if (old.label != widget.label) lifecycle.add('didUpdateWidget ${old.label}->${widget.label}');
  }

  @override
  void dispose() {
    lifecycle.add('dispose ${widget.label}');
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => TextButton(
        onPressed: () => setState(() => taps++),
        child: Text('${widget.label}: $taps'),
      );
}
```

```dart title="element_test.dart"
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:l2/a/element_demo.dart';

Widget host(Widget child) => MaterialApp(home: Scaffold(body: child));
List<String> labels(WidgetTester t) =>
    [for (final w in t.widgetList<Text>(find.descendant(of: find.byType(TextButton), matching: find.byType(Text)))) w.data!];

void main() {
  setUp(lifecycle.clear);

  testWidgets('1. same type, same position: element and State are reused', (tester) async {
    await tester.pumpWidget(host(const Tally(label: 'A')));
    await tester.tap(find.byType(TextButton));
    await tester.pump();
    await tester.pumpWidget(host(const Tally(label: 'B')));
    print('${labels(tester)}  lifecycle=$lifecycle');
  });

  testWidgets('2. wrapping in another widget changes the position: State is lost', (tester) async {
    await tester.pumpWidget(host(const Tally(label: 'A')));
    await tester.tap(find.byType(TextButton));
    await tester.pump();
    await tester.pumpWidget(host(const Padding(padding: EdgeInsets.all(8), child: Tally(label: 'A'))));
    print('${labels(tester)}  lifecycle=$lifecycle');
  });

  testWidgets('3a. reordering without keys: state stays with the slot', (tester) async {
    await tester.pumpWidget(host(const Column(children: [Tally(label: 'X'), Tally(label: 'Y')])));
    await tester.tap(find.text('X: 0'));
    await tester.pump();
    print('before swap ${labels(tester)}');
    await tester.pumpWidget(host(const Column(children: [Tally(label: 'Y'), Tally(label: 'X')])));
    print('after swap  ${labels(tester)}  lifecycle=$lifecycle');
  });

  testWidgets('3b. reordering with keys: state moves with the widget', (tester) async {
    await tester.pumpWidget(host(const Column(children: [Tally(key: ValueKey('X'), label: 'X'), Tally(key: ValueKey('Y'), label: 'Y')])));
    await tester.tap(find.text('X: 0'));
    await tester.pump();
    await tester.pumpWidget(host(const Column(children: [Tally(key: ValueKey('Y'), label: 'Y'), Tally(key: ValueKey('X'), label: 'X')])));
    print('after swap  ${labels(tester)}  lifecycle=$lifecycle');
  });

  testWidgets('4. GlobalKey: state survives moving to a different parent', (tester) async {
    final key = GlobalKey();
    await tester.pumpWidget(host(Tally(key: key, label: 'G')));
    await tester.tap(find.byType(TextButton));
    await tester.pump();
    await tester.pumpWidget(host(Padding(padding: const EdgeInsets.all(8), child: Tally(key: key, label: 'G'))));
    print('${labels(tester)}  lifecycle=$lifecycle');
  });

  testWidgets('5. BuildContext is the Element', (tester) async {
    await tester.pumpWidget(host(Builder(builder: (context) {
      final element = context as Element;
      print('context is ${element.runtimeType}, widget ${element.widget.runtimeType}, depth ${element.depth}');
      final chain = <String>[];
      context.visitAncestorElements((e) {
        chain.add(e.widget.runtimeType.toString());
        return chain.length < 6;
      });
      print('nearest ancestors: $chain');
      return const SizedBox();
    })));
  });
}
```

```console
$ flutter test test/a/element_test.dart
[B: 1]  lifecycle=[initState A, didUpdateWidget A->B]
[A: 0]  lifecycle=[initState A, initState A, dispose A]
before swap [X: 1, Y: 0]
after swap  [Y: 1, X: 0]  lifecycle=[initState X, initState Y, didUpdateWidget X->Y, didUpdateWidget Y->X]
after swap  [Y: 0, X: 1]  lifecycle=[initState X, initState Y]
[G: 1]  lifecycle=[initState G]
context is StatelessElement, widget Builder, depth 155
nearest ancestors: [KeyedSubtree, _BodyBuilder, MediaQuery, LayoutId, CustomMultiChildLayout, _ActionsScope]
00:00 +6: All tests passed!
```

**1. Same type, same position → reused.** Changing the label from `A` to `B` kept the tap count (`B: 1`). There was no
second `initState`, only `didUpdateWidget`: the element stayed, its `State` stayed, and it was given the new widget.
This is why `setState` is cheap and why state survives rebuilds at all.

**2. Wrapping changes the position → state lost.** Inserting a `Padding` meant the slot that used to hold a `Tally` now
holds a `Padding`. Different type, so the old element was unmounted (`dispose A`) and a new `Tally` built from scratch
inside the padding (`A: 0`). Conditionally wrapping a widget (`if (wide) Padding(child: x) else x`) is the classic way to
lose a text field's contents or a scroll position. Keep the structure stable (e.g. `Padding(padding: wide ? ... :
EdgeInsets.zero, child: x)`) or use a `GlobalKey`.

**3a. Reordering without keys → state stays in the slot.** Both children are `Tally` with no key, so `canUpdate` is true
position by position. Flutter simply gave the first element the `Y` widget and the second the `X` widget. The count `1`
stayed in the *first slot* and now shows next to `Y`. The user tapped X; the UI now claims they tapped Y.

**3b. Reordering with keys → state moves.** With `ValueKey`s, Flutter matches children by key within the same parent,
moves the elements, and the count follows `X`. No `didUpdateWidget` calls were needed because the widgets didn't change.
Rule: **give stateful children of a list a key derived from the data's identity** (an id, not the index).

**4. GlobalKey → state survives reparenting.** A `GlobalKey` is unique across the whole app. When a widget with a global
key disappears from one place and appears in another in the same frame, Flutter moves the element (and its `State` and
render object) instead of recreating it. Use sparingly: global keys are more expensive and are the reason
`Form`'s `GlobalKey<FormState>` can reach into a form's state from outside.

**5. `BuildContext` is the element.** `context` passed to `build` *is* the widget's `Element` (the interface exists to
keep you from calling element internals). Its depth was 155 under a plain `MaterialApp` + `Scaffold` — a reminder of how
many widgets the framework adds around yours (`MediaQuery`, `_ActionsScope`, layout widgets…). `Theme.of(context)`,
`Navigator.of(context)` and friends start from *this* element and look upward, which is why the context you pass matters.

## Practical consequences

- **"Looking up a deactivated widget's ancestor is unsafe"**: you used a `context` after its element was unmounted —
  typically after an `await`. Check `context.mounted` (or `mounted` in a `State`).
- **`context` from the wrong level**: `Scaffold.of(context)` from the same `build` that creates the `Scaffold` fails,
  because that context is *above* the scaffold. A `Builder` or an extracted widget gives you a context below it.
- **Keys on lists** of stateful items or items with animations: `ValueKey(item.id)`. `UniqueKey()` in `build` makes a
  new key every frame, so the element is recreated every frame — almost never what you want.
- **`const` constructors** let Flutter skip work: an identical widget instance at the same position needn't be rebuilt.

## How It Actually Works

When a `StatefulElement` is marked dirty (by `setState`), the `BuildOwner` adds it to a dirty list. During the next frame's
build phase, each dirty element runs `build()` and then calls `updateChild(oldChild, newWidget, slot)` for its child. That
method is the heart of the framework:

- new widget is `null` → deactivate the old child;
- old child exists and `Widget.canUpdate(old.widget, newWidget)` → `oldChild.update(newWidget)` (for stateful elements this
  calls `didUpdateWidget` and rebuilds);
- otherwise → deactivate the old child and **inflate** a new element from the new widget (`createElement`, `mount`, and for
  stateful widgets `createState` and `initState`).

For multi-child widgets (`Column`, `ListView`), `updateChildren` first matches unkeyed children from the top and bottom of
the list, then puts the remaining old children with keys into a map and looks each new keyed widget up in it — that's how
keyed children are found again after a reorder. Deactivated elements are held until the end of the frame; if a widget with
the same `GlobalKey` appears elsewhere before then, the element is reactivated in its new location. Otherwise it's unmounted
and `dispose` runs.

Render objects follow elements: `RenderObjectElement`s create a render object when mounted and update its properties in
`updateRenderObject` when given a new widget, so a reused element means a reused render object and no re-layout unless a
property actually changed. [Level 4 · 01](../level-4/01-rendering-pipeline.md) picks up from there.

## Common mistakes

- **Index keys** (`ValueKey(index)`) on reorderable lists — the key moves with the slot, so it's no better than no key.
- **Conditional wrappers** that change the tree shape and silently reset state.
- **`UniqueKey()` created in `build`.**
- **Storing a `BuildContext` in a field** and using it later; the element may be gone.
- **Putting a `GlobalKey` in a `build` method** (`final key = GlobalKey();` inside `build`) — a new key every build means
  a new element every build.

## Exercise

1. Re-run experiment 3a with a `TextField` inside each tile instead of a counter. Type into the first, swap, and observe.
   Fix it with keys.
2. In experiment 2, keep the state *without* a `GlobalKey` by changing how the padding is applied.
3. Add a `ListView` of 50 `Tally` tiles and scroll until the first is off-screen, then back. Is the count still there? Read
   about `AutomaticKeepAliveClientMixin` and make it survive.
4. Use `debugDumpApp()` in a running app and find your own widgets in the output. How deep are they?
