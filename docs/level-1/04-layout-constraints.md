---
description: "Flutter layout by measurement: constraints down, sizes up, why SizedBox can be ignored, Row/Column/Expanded, and the two classic layout errors."
---

# 04 · Layout: Constraints Go Down, Sizes Go Up

Flutter layout has one rule, and almost every confusing layout bug is that rule in disguise:

> **Constraints go down. Sizes go up. Parents set positions.**

A parent tells each child the minimum and maximum width and height it may be (its
**constraints**). The child picks a size within them and reports it back. The parent then
decides where to put the child. A child can't choose its own position, and it can't be bigger
than its constraints allow — no matter what `width:` you wrote. This lesson makes that concrete
by *printing* the constraints and sizes in real layouts, including the two error messages every
Flutter developer meets in their first week.

## A measuring harness

```dart title="layout_test.dart"
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

/// Pumps [child] at the root of an 800x600 test screen and reports
/// the constraints it received and the size it chose.
Future<void> probe(WidgetTester tester, String label, Widget child) async {
  final key = GlobalKey();
  await tester.pumpWidget(Directionality(
    textDirection: TextDirection.ltr,
    child: KeyedSubtree(key: key, child: child),
  ));
  // Measure the innermost box: the blue ColoredBox.
  final target = tester.renderObject<RenderBox>(find.byType(ColoredBox));
  final err = tester.takeException();
  print('${label.padRight(44)} constraints=${target.constraints}  size=${target.size}'
      '${err != null ? '  EXCEPTION: ${err.toString().split('\n').first}' : ''}');
}

void main() {
  testWidgets('constraints go down, sizes go up', (tester) async {
    const blue = ColoredBox(color: Colors.blue);
    await probe(tester, 'root: SizedBox(100x100)', const SizedBox(width: 100, height: 100, child: blue));
    await probe(tester, 'root: Center > SizedBox(100x100)',
        const Center(child: SizedBox(width: 100, height: 100, child: blue)));
    await probe(tester, 'Center > SizedBox(2000x50)',
        const Center(child: SizedBox(width: 2000, height: 50, child: blue)));
    await probe(tester, 'Center > ConstrainedBox(min 300x300) > 100x100',
        Center(child: ConstrainedBox(constraints: const BoxConstraints(minWidth: 300, minHeight: 300),
            child: SizedBox(width: 100, height: 100, child: blue))));
    await probe(tester, 'Center > UnconstrainedBox > 2000x50',
        const Center(child: UnconstrainedBox(child: SizedBox(width: 2000, height: 50, child: blue))));
  });

  testWidgets('what each child is told', (tester) async {
    final seen = <String, BoxConstraints>{};
    Widget spy(String name, Widget child) => LayoutBuilder(builder: (context, c) {
          seen[name] = c;
          return child;
        });

    await tester.pumpWidget(Directionality(
      textDirection: TextDirection.ltr,
      child: spy('root', Center(
        child: spy('inside Center', Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            spy('Column child', const SizedBox(width: 50, height: 20)),
            SizedBox(width: 300, child: Row(children: [
              spy('Row child', const SizedBox(width: 40, height: 20)),
              Expanded(child: spy('Expanded in Row', const SizedBox(height: 20))),
            ])),
            SizedBox(height: 100, child: spy('ListView parent', const SizedBox())),
          ],
        )),
      )),
    ));
    seen.forEach((k, v) => print('${k.padRight(18)} $v'));
  });

  testWidgets('the classic overflow', (tester) async {
    await tester.pumpWidget(const Directionality(
      textDirection: TextDirection.ltr,
      child: Center(
        child: SizedBox(
          width: 200,
          child: Row(children: [
            Text('A very long title that does not fit in two hundred pixels'),
          ]),
        ),
      ),
    ));
    final e = tester.takeException();
    print('Row with long Text: ${e.toString().split('\n').first}');

    await tester.pumpWidget(const Directionality(
      textDirection: TextDirection.ltr,
      child: Center(
        child: SizedBox(
          width: 200,
          child: Row(children: [
            Expanded(child: Text('A very long title that does not fit in two hundred pixels',
                overflow: TextOverflow.ellipsis)),
          ]),
        ),
      ),
    ));
    print('with Expanded + ellipsis: exception=${tester.takeException()}');
  });

  testWidgets('unbounded height', (tester) async {
    await tester.pumpWidget(Directionality(
      textDirection: TextDirection.ltr,
      child: Column(children: [
        ListView(children: const [Text('a'), Text('b')]),
      ]),
    ));
    final e = tester.takeException();
    print('ListView in Column: ${e.toString().split('\n').first}');
  });

  testWidgets('bounded fix', (tester) async {
    await tester.pumpWidget(Directionality(
      textDirection: TextDirection.ltr,
      child: Column(children: [
        Expanded(child: ListView(children: const [Text('a'), Text('b')])),
      ]),
    ));
    print('ListView in Expanded in Column: exception=${tester.takeException()}');
  });
}
```

Two tools do the measuring. `probe` pumps a widget at the root of the 800 × 600 test screen and
reads the innermost `ColoredBox`'s render object: its `constraints` (what it was told) and its
`size` (what it chose). `spy` wraps a `LayoutBuilder` around a child, which receives the
constraints that position in the tree is given.

## Experiment 1: when your `SizedBox` is ignored

```text
root: SizedBox(100x100)                      constraints=BoxConstraints(w=800.0, h=600.0)  size=Size(800.0, 600.0)
root: Center > SizedBox(100x100)             constraints=BoxConstraints(w=100.0, h=100.0)  size=Size(100.0, 100.0)
Center > SizedBox(2000x50)                   constraints=BoxConstraints(w=800.0, h=50.0)  size=Size(800.0, 50.0)
Center > ConstrainedBox(min 300x300) > 100x100 constraints=BoxConstraints(w=300.0, h=300.0)  size=Size(300.0, 300.0)
Center > UnconstrainedBox > 2000x50          constraints=BoxConstraints(w=2000.0, h=50.0)  size=Size(2000.0, 50.0)  EXCEPTION: A RenderConstraintsTransformBox overflowed by 600 pixels on the left and 600 pixels on the right.
```

1. **At the root, a 100 × 100 `SizedBox` filled the whole screen.** The screen gives its child
   **tight** constraints — `w=800, h=600` means minimum and maximum are both exactly that. The
   `SizedBox` asked for 100 × 100, but a child must satisfy its constraints, so it became 800 × 600.
2. **Wrapped in `Center`, it's 100 × 100.** `Center` takes the tight screen constraints for itself
   and passes **loose** ones down (`0 ≤ w ≤ 800, 0 ≤ h ≤ 600`). Now the box's request fits.
3. **2000 pixels wide isn't possible.** Loose constraints still have a maximum; the `SizedBox` was
   clamped to 800 wide.
4. **A minimum wins over a request.** `ConstrainedBox(minWidth: 300, minHeight: 300)` made the
   child's constraints `300..800`, so the 100 × 100 request became 300 × 300.
5. **`UnconstrainedBox` removes the limits** — the child really is 2000 pixels wide — and the
   overflow is reported, because the parent can't actually show it.

Every "why is my widget the wrong size?" question is answered by asking *what constraints did its
parent give it?*

## Experiment 2: what Center, Column, Row and Expanded pass down

```text
root               BoxConstraints(w=800.0, h=600.0)
inside Center      BoxConstraints(0.0<=w<=800.0, 0.0<=h<=600.0)
Column child       BoxConstraints(0.0<=w<=800.0, 0.0<=h<=Infinity)
Row child          BoxConstraints(unconstrained)
Expanded in Row    BoxConstraints(w=260.0, 0.0<=h<=Infinity)
ListView parent    BoxConstraints(0.0<=w<=800.0, h=100.0)
```

- **`Column` gives children unbounded height** (`h ≤ Infinity`). A column lays its children out
  one after another and adds up their heights, so it can't tell any one child how much is left —
  it asks each child how tall it wants to be. Width stays bounded by the column's own width.
- **`Row` does the same horizontally** for non-flexible children. Here the row itself sat inside the
  column, so its child was unbounded in *both* directions.
- **`Expanded` gets a tight width**: the row was 300 wide, the fixed child took 40, so the
  `Expanded` child is told *exactly* 260. That's how `Expanded` "fills the remaining space": the row
  measures the inflexible children first, then divides what's left among flexible ones by their
  `flex` factors.
- **`SizedBox(height: 100)` makes height tight** — useful for giving a scrolling list a bounded box.

## Error 1: "A RenderFlex overflowed"

```text
Row with long Text: A RenderFlex overflowed by 598 pixels on the right.
with Expanded + ellipsis: exception=null
```

`Row` gives its `Text` unbounded width, so the text lays out on one line at its natural width —
far wider than 200 pixels — and the row overflows. (On a device this shows the yellow-and-black
striped warning in debug mode.) The fix is to *bound* the text: wrap it in `Expanded` (or
`Flexible`), which hands it a tight width it must wrap or truncate within, and choose
`overflow: TextOverflow.ellipsis` or let it wrap onto several lines. "Overflowed by N pixels" almost
always means a child inside a `Row`/`Column` needs `Expanded` or `Flexible`.

## Error 2: "Vertical viewport was given unbounded height"

Put a `ListView` directly in a `Column` and the first exception is:

```text
The following assertion was thrown during performResize():
Vertical viewport was given unbounded height.
Viewports expand in the scrolling direction to fill their container. In this case, a vertical
viewport was given an unlimited amount of vertical space in which to expand. This situation
typically happens when a scrollable widget is nested inside another scrollable widget.
```

followed by a cascade (14 exceptions in this test). A `ListView` wants to be as tall as its parent
allows, so it can scroll within that height; a `Column` allows infinity. Nothing can be infinitely
tall. Fixes, depending on intent:

- **`Expanded(child: ListView(...))`** — the list takes the rest of the column. This is the usual
  answer, and it ran without exceptions above.
- **`SizedBox(height: 200, child: ListView(...))`** — a fixed-height scrolling region.
- **`shrinkWrap: true`** — the list sizes itself to its content. Fine for short lists, but it lays out
  every item, defeating lazy building for long ones ([lesson 07](07-lists-keys.md)).

## A layout checklist

When something is the wrong size or overflows:

1. Find the widget's parent chain up to the nearest `Scaffold` body or screen.
2. For each parent, ask: tight, loose or unbounded — in which direction?
3. Inside `Row`/`Column`, give flexible children `Expanded`/`Flexible`.
4. Never put an unbounded-in-the-scroll-direction widget (list, grid, `Expanded`) where the
   constraint is infinite.
5. Use a `LayoutBuilder` (or the DevTools layout explorer) to print constraints instead of guessing.

## How It Actually Works

Layout is performed by **render objects**, not widgets. A `RenderBox`'s parent calls
`child.layout(constraints, parentUsesSize: true)`. The child's `performLayout` lays out *its*
children the same way, then sets `size`, which must satisfy `constraints` (in debug mode an
assertion checks it). The parent reads `child.size` and stores an offset for the child in the
child's `parentData` — that's "parents set positions". Each render object is laid out once per
frame in a single pass down and up the tree, which is why Flutter layout is linear in the number
of render objects, unlike layout systems that measure children several times.

`RenderFlex` (behind `Row` and `Column`) lays out inflexible children with unbounded main-axis
constraints, sums their sizes, subtracts from its own maximum, and lays out `Flexible`/`Expanded`
children with tight (or loose, for `Flexible`) shares of the remainder. If the inflexible children
alone exceed the maximum, it paints them clipped and reports the overflow. `Center` is a
`RenderPositionedBox`: it loosens constraints for its child, sizes itself as large as allowed, and
positions the child in the middle.

## Common mistakes

- **Setting `width`/`height` and expecting it to win** against tight parent constraints.
- **Long `Text` in a `Row` without `Expanded`/`Flexible`.**
- **Scrollables inside `Column`s** without bounding them.
- **Reaching for `shrinkWrap: true` by default**, quietly turning lazy lists eager.
- **Wrapping everything in `Container`** with many properties — use the focused widgets (`Padding`,
  `SizedBox`, `ColoredBox`, `Align`) so it's clear which constraint does what.

## Exercise

1. Add `probe(tester, 'Align(topLeft) > 100x100', ...)` and predict the constraints and size first.
2. Put two `Expanded` children with `flex: 2` and `flex: 1` in a 300-wide row and print their widths.
3. Replace `Expanded` with `Flexible` around the long text. What changes in the constraints its child
   receives, and does the overflow come back?
4. Make the `ListView` work inside the column with `shrinkWrap: true`, then explain why it's a poor
   choice for a list of 10,000 rows.
