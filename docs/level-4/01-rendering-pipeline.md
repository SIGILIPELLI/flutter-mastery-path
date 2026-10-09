---
description: "Inside Flutter's frame — build, layout, paint, composite, semantics — with a custom RenderBox that measures what triggers layout vs paint."
---

# 01 · The Rendering Pipeline & RenderObjects

Everything in Flutter eventually becomes a **render object**: a node that knows its size, where its children go, how to
paint, and how to respond to hit tests. Widgets are configuration; elements are identity
([Level 3 · 01](../level-3/01-element-tree.md)); render objects do the work. Knowing this layer explains most performance
advice — "use `const`", "add a `RepaintBoundary`", "avoid intrinsic sizing" — and lets you build components no stock widget
can. This lesson writes one from scratch and counts what it does.

## One frame, five phases

When the engine signals vsync, `WidgetsBinding.drawFrame` runs, in order:

| Phase | Who's involved | What happens |
|---|---|---|
| **Animate** | tickers | animation controllers advance ([Level 3 · 02](../level-3/02-animations.md)) |
| **Build** | dirty elements | `build()` runs for elements marked by `setState` & co; widgets diffed against elements |
| **Layout** | render objects needing layout | constraints go down, sizes come up, parents position children |
| **Paint** | render objects needing paint | drawing commands recorded into layers |
| **Composite** + **semantics** | layer tree, semantics owner | layer tree sent to the engine for rasterization; accessibility tree updated |

Each phase only visits *dirty* nodes. The whole game of Flutter performance is keeping those dirty sets small.

## A custom RenderBox

A progress bar whose height depends on `thickness` (layout) and whose fill depends on `value` (paint only):

```dart title="render_demo.dart"
import 'package:flutter/rendering.dart';
import 'package:flutter/widgets.dart';

/// Widget: immutable configuration. Creates and updates the render object.
class ProgressBar extends LeafRenderObjectWidget {
  const ProgressBar({super.key, required this.value, this.thickness = 8, this.color = const Color(0xFF3F51B5)});
  final double value; // 0..1
  final double thickness;
  final Color color;

  @override
  RenderProgressBar createRenderObject(BuildContext context) =>
      RenderProgressBar(value: value, thickness: thickness, color: color, textDirection: Directionality.of(context));

  @override
  void updateRenderObject(BuildContext context, RenderProgressBar renderObject) {
    renderObject
      ..value = value
      ..thickness = thickness
      ..color = color
      ..textDirection = Directionality.of(context);
  }
}

/// RenderObject: does layout and paint. Each setter requests only the work it needs.
class RenderProgressBar extends RenderBox {
  RenderProgressBar({required double value, required double thickness, required Color color, required TextDirection textDirection})
      : _value = value,
        _thickness = thickness,
        _color = color,
        _textDirection = textDirection;

  static int layouts = 0;
  static int paints = 0;

  double _value;
  set value(double v) {
    if (v == _value) return;
    _value = v;
    markNeedsPaint(); // the size doesn't depend on value
    markNeedsSemanticsUpdate();
  }

  double _thickness;
  set thickness(double t) {
    if (t == _thickness) return;
    _thickness = t;
    markNeedsLayout(); // height depends on thickness
  }

  Color _color;
  set color(Color c) {
    if (c == _color) return;
    _color = c;
    markNeedsPaint();
  }

  // Semantics labels need a reading direction (and an RTL app should fill from the right — exercise 1).
  TextDirection _textDirection;
  set textDirection(TextDirection d) {
    if (d == _textDirection) return;
    _textDirection = d;
    markNeedsSemanticsUpdate();
  }

  @override
  void performLayout() {
    layouts++;
    // Take all the width offered (or a default if unbounded); height is our thickness.
    final width = constraints.hasBoundedWidth ? constraints.maxWidth : 200.0;
    size = constraints.constrain(Size(width, _thickness));
  }

  @override
  double computeMinIntrinsicHeight(double width) => _thickness;
  @override
  double computeMaxIntrinsicHeight(double width) => _thickness;

  @override
  void paint(PaintingContext context, Offset offset) {
    paints++;
    final canvas = context.canvas;
    final track = offset & size;
    final radius = Radius.circular(size.height / 2);
    canvas.drawRRect(RRect.fromRectAndRadius(track, radius), Paint()..color = _color.withValues(alpha: 0.2));
    final filled = Rect.fromLTWH(offset.dx, offset.dy, size.width * _value.clamp(0, 1), size.height);
    canvas.drawRRect(RRect.fromRectAndRadius(filled, radius), Paint()..color = _color);
  }

  @override
  bool hitTestSelf(Offset position) => true;

  @override
  void describeSemanticsConfiguration(SemanticsConfiguration config) {
    super.describeSemanticsConfiguration(config);
    config
      ..textDirection = _textDirection
      ..label = 'Progress'
      ..value = '${(_value * 100).round()}%';
  }
}
```

The pattern every render-object widget follows:

1. A `RenderObjectWidget` (`LeafRenderObjectWidget` for no children, `SingleChildRenderObjectWidget`,
   `MultiChildRenderObjectWidget`) with `createRenderObject` and `updateRenderObject`.
2. A `RenderBox` whose **setters compare and mark the minimum work**: `value` and `color` only affect pixels →
   `markNeedsPaint()`; `thickness` affects size → `markNeedsLayout()` (which implies paint).
3. `performLayout` reads `constraints` and sets `size` — never anything else's size, and always within the constraints
   (`constraints.constrain`).
4. `paint` draws at `offset` (the object's position in the current layer — not 0,0).
5. Intrinsics, hit testing and semantics, so it behaves like a good citizen.

## Measuring the phases

```dart title="render_test.dart"
import 'package:flutter/material.dart';
import 'package:flutter/rendering.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:l2/m/render_demo.dart';

void main() {
  Widget app(double value, double thickness, {String label = 'x'}) => MaterialApp(
        home: Scaffold(
          body: Column(children: [
            Text(label),
            Padding(padding: const EdgeInsets.all(16), child: ProgressBar(value: value, thickness: thickness)),
          ]),
        ),
      );

  testWidgets('which setter triggers which phase', (tester) async {
    RenderProgressBar.layouts = 0;
    RenderProgressBar.paints = 0;
    String counts() => 'layouts=${RenderProgressBar.layouts} paints=${RenderProgressBar.paints}';

    await tester.pumpWidget(app(0.25, 8));
    print('first frame:          ${counts()}  size=${tester.getSize(find.byType(ProgressBar))}');
    await tester.pumpWidget(app(0.25, 8));
    print('identical rebuild:    ${counts()}');
    await tester.pumpWidget(app(0.75, 8));
    print('value 0.25 -> 0.75:   ${counts()}');
    await tester.pumpWidget(app(0.75, 16));
    print('thickness 8 -> 16:    ${counts()}  size=${tester.getSize(find.byType(ProgressBar))}');
    await tester.pumpWidget(app(0.75, 16, label: 'a much longer label'));
    print('sibling text changed: ${counts()}');
  });

  testWidgets('render tree and semantics', (tester) async {
    final handle = tester.ensureSemantics();
    await tester.pumpWidget(app(0.4, 8));
    final ro = tester.renderObject<RenderProgressBar>(find.byType(ProgressBar));
    print('constraints: ${ro.constraints}');
    print('parent chain: ${[for (RenderObject? p = ro.parent; p != null && p is! RenderView; p = p.parent) p.runtimeType].take(4).toList()}');
    final node = tester.getSemantics(find.byType(ProgressBar));
    print('semantics: label="${node.label}" value="${node.value}"');
    handle.dispose();
  });
}
```

```console
$ flutter test test/m/render_test.dart
first frame:          layouts=1 paints=1  size=Size(768.0, 8.0)
identical rebuild:    layouts=1 paints=1
value 0.25 -> 0.75:   layouts=1 paints=2
thickness 8 -> 16:    layouts=2 paints=3  size=Size(768.0, 16.0)
sibling text changed: layouts=2 paints=4
constraints: BoxConstraints(0.0<=w<=768.0, 0.0<=h<=Infinity)
parent chain: [RenderPadding, RenderFlex, RenderCustomMultiChildLayoutBox, _RenderInkFeatures]
semantics: label="Progress" value="40%"
00:00 +2: All tests passed!
```

- **Identical rebuild: zero work.** The widget was rebuilt, `updateRenderObject` called every setter, but every setter saw an
  equal value and returned early. That early return is why rebuilding widgets is cheap *if* render objects are written well.
- **Value change: paint only.** No layout.
- **Thickness change: layout + paint**, and the new size is 16 high.
- **Sibling text changed: paint, no layout.** The `Column` re-laid out its children, but asked our bar to lay out with the
  *same constraints* it had before, and a render object that isn't dirty and receives identical constraints skips
  `performLayout`. It still repainted because it shares a layer with the text that changed (the same effect as in
  [Level 3 · 03](../level-3/03-custom-painter.md)).
- **Constraints** show what the `Column` + `Padding` offered: width up to 768 (800 minus 2×16 padding), height unbounded —
  which is why `performLayout` takes its height from `thickness` rather than from `constraints.maxHeight`.

## A bug we hit: semantics need a direction

The first version set `config.label = 'Progress'` without a text direction, and the semantics test failed with:

```console
A SemanticsData object with label "Progress" had a null textDirection.
Failed assertion: line 1089 pos 10: 'attributedLabel.string == '' || textDirection != null'
```

Screen readers need to know the reading direction of any label. Stock widgets get it from the ambient `Directionality`; a
custom render object has to be given it. The fix — read `Directionality.of(context)` in `createRenderObject`/
`updateRenderObject` and pass it into the semantics configuration — is in the code above. It's a good example of
responsibilities a stock widget handles silently that you take on when you drop down a layer.

## Layout rules worth memorizing

- **Constraints down, sizes up, parent sets position.** A child can't choose its position; a parent can't choose its child's
  size except through constraints.
- **Relayout boundaries**: when a render object's size can't change its parent's layout — because its constraints are tight,
  or it's `sizedByParent`, or the parent doesn't use its size — a `markNeedsLayout` stops propagating at it. Otherwise,
  layout dirtiness walks up the tree.
- **Intrinsic sizing is expensive**: `IntrinsicHeight`/`IntrinsicWidth` ask the subtree "how big would you like to be?"
  before laying it out, which can mean laying out (or measuring) twice — O(n²) when nested. Avoid it in lists.
- **Paint order is child order**; hit testing goes in reverse (topmost first).

## When to write a RenderObject

Rarely. Try composition first, then `CustomPaint` (paint-only custom visuals), then `CustomMultiChildLayout`/`Flow`
(custom positioning of widget children), then `CustomSingleChildLayout`. Write a render object when you need custom
**layout and** custom paint and hit testing together, or when profiling shows a composed widget doing too much work — for
example, a chat bubble that sizes itself to its text's last line, or a text-heavy timeline with thousands of items.

## How It Actually Works

`markNeedsLayout` adds the render object (or its relayout boundary) to `PipelineOwner._nodesNeedingLayout`;
`markNeedsPaint` adds it (or its nearest repaint boundary) to `_nodesNeedingPaint`, and both request a frame.
In `drawFrame`, after build, `flushLayout` sorts dirty nodes by depth and calls `layout(constraints)` on each, which calls
`performLayout` only if the node is dirty or the constraints changed. `flushCompositingBits` updates which subtrees need
their own compositing layers, `flushPaint` repaints dirty repaint boundaries into fresh `PictureLayer`s, and
`flushSemantics` rebuilds changed semantics nodes. Finally `renderView.compositeFrame()` builds a `Scene` from the layer
tree and hands it to the engine, whose raster thread rasterizes it with Impeller or Skia on the GPU while the UI thread
starts on the next frame. Two threads is why the performance overlay shows two graphs — UI and raster —
which the next lesson uses.

## Common mistakes

- **Setters that always mark dirty** (no equality check) — every rebuild becomes a relayout.
- **`markNeedsLayout` for paint-only properties.**
- **Setting `size` outside the constraints** — caught by an assertion in debug mode.
- **Painting at `Offset.zero` instead of `offset`** — your object draws at the top-left of the layer.
- **Forgetting semantics and hit testing**, producing a component that looks right but is invisible to screen readers or
  unclickable.

## Exercise

1. Make the bar fill from the right when `textDirection` is RTL. Add a golden test for both directions.
2. Add an `indeterminate` mode driven by an `Animation<double>` passed to the render object, which calls `markNeedsPaint` on
   each tick. Count layouts during a second of animation — it should be zero.
3. Implement `computeDryLayout` so the bar works inside widgets that ask for dry layout (for example, inside some `Wrap` or
   `IntrinsicHeight` cases).
4. Wrap the bar in `RepaintBoundary` in the test and confirm "sibling text changed" no longer paints it.
