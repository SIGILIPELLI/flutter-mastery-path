---
description: "What Flutter really is: Dart code, a rendering engine you ship with your app, and three trees — widgets, elements, render objects — seen in a real test."
---

# 01 · What Flutter Is: Engine, Widgets & One Codebase

Flutter is a UI toolkit for building apps for Android, iOS, the web, Windows, macOS and Linux
from one Dart codebase. Plenty of tools promise "one codebase"; what makes Flutter different
is *how*. It doesn't wrap the platform's native buttons and text fields. It ships its own
rendering engine inside your app and draws every pixel itself — the same way a game engine
does. That one design decision explains most of what you'll experience: identical UI on every
platform, smooth animation, a large app binary, and some work to make things feel "native"
when you want them to.

This lesson gives you the mental model the rest of the course builds on, and proves it with a
real test rather than a diagram.

## The layers

```text
┌──────────────────────────────────────────────┐
│ Your app (Dart): widgets, state, logic       │
├──────────────────────────────────────────────┤
│ Framework (Dart): Material, Cupertino,       │
│ widgets, rendering, gestures, animation      │
├──────────────────────────────────────────────┤
│ Engine (C++): Impeller/Skia graphics, text   │
│ layout, Dart runtime                         │
├──────────────────────────────────────────────┤
│ Embedder: Android, iOS, web, desktop shells  │
└──────────────────────────────────────────────┘
```

- **Your app and the framework are both Dart.** You can step into `Text`, `Row` or
  `MaterialApp` in your editor and read exactly what they do. Nothing above the engine is a
  black box.
- **The engine** turns the framework's drawing commands into GPU work and handles text shaping.
  On iOS and recent Android, Flutter renders with **Impeller**, its newer graphics backend;
  Skia is still used on some platforms.
- **The embedder** is the thin platform-specific shell: it creates a surface to draw on, forwards
  touches and keyboard input, and gives Dart a way to call platform APIs (platform channels,
  [Level 3 · 05](../level-3/05-platform-channels.md)).

In release mode, Dart is compiled ahead-of-time to native machine code (or to JavaScript /
WebAssembly on the web). In debug mode it runs on a VM with a JIT compiler, which is what makes
**hot reload** possible: change code, and the running app updates in about a second without
losing its state.

## A minimal app

You'll set up projects properly in [lesson 02](02-setup-first-app.md). Here's the smallest
useful app — no Material design, just the base widgets library:

```dart title="bare.dart"
import 'package:flutter/widgets.dart';

// The smallest useful Flutter app: no Material, no Cupertino — just widgets.
void main() => runApp(const Greeting());

class Greeting extends StatelessWidget {
  const Greeting({super.key});

  @override
  Widget build(BuildContext context) {
    return const Directionality(
      textDirection: TextDirection.ltr,
      child: Center(
        child: Text('Hello, Flutter', style: TextStyle(fontSize: 32)),
      ),
    );
  }
}
```

Everything on screen is a **widget**: `Greeting` (yours), `Directionality` (tells text which way
to flow), `Center` and `Text`. A widget is an immutable *description* of part of the UI. `build`
returns a new description whenever Flutter asks for one; it doesn't draw anything itself.

## Three trees, observed

Flutter keeps three parallel structures. This test pumps the app in Flutter's headless test
environment and walks them:

```dart title="trees_test.dart"
import 'package:flutter/rendering.dart';
import 'package:flutter/widgets.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hello/l1/bare.dart';

void main() {
  testWidgets('one widget description, three trees', (tester) async {
    await tester.pumpWidget(const Greeting());

    // 1. Widgets: immutable descriptions we wrote.
    final widgets = <String>[];
    void visit(Element e, int depth) {
      widgets.add('${'  ' * depth}${e.widget.runtimeType}'
          '${e.renderObject != null && e is RenderObjectElement ? '  -> ${e.renderObject.runtimeType}' : ''}');
      e.visitChildren((c) => visit(c, depth + 1));
    }
    visit(tester.element(find.byType(Greeting)), 0);
    print(widgets.join('\n'));

    // 2. Render objects: what actually lays out and paints.
    final paragraph = tester.renderObject<RenderParagraph>(find.text('Hello, Flutter'));
    print('text size: ${paragraph.size}');
    print('screen size: ${tester.view.physicalSize / tester.view.devicePixelRatio}');

    // 3. Rebuilding with a new widget reuses the same element and render object.
    final before = tester.element(find.text('Hello, Flutter'));
    await tester.pumpWidget(const Greeting());
    final after = tester.element(find.text('Hello, Flutter'));
    print('same element after rebuild: ${identical(before, after)}');
    expect(find.text('Hello, Flutter'), findsOneWidget);
  });
}
```

```console
$ flutter test test/l1/trees_test.dart
00:00 +0: one widget description, three trees
Greeting
  Directionality
    Center  -> RenderPositionedBox
      Text
        RichText  -> RenderParagraph
text size: Size(448.0, 32.0)
screen size: Size(800.0, 600.0)
same element after rebuild: true
00:00 +1: All tests passed!
```

(Flutter 3.44.8, Dart 3.12.2.) Three things are visible here:

1. **Widgets** — the names on the left. Notice `RichText` under `Text`: you never wrote it.
   `Text` is itself a widget whose `build` returns a `RichText`. Most widgets are compositions of
   other widgets, all the way down.
2. **Elements** — the walk itself visits *elements*, the objects Flutter creates for each widget
   and keeps alive between frames. Each element holds a reference to its current widget and its
   place in the tree.
3. **Render objects** — only some widgets create one (shown with `->`). `Center` creates a
   `RenderPositionedBox`; `RichText` creates a `RenderParagraph`, which actually measures and
   paints text. `Greeting`, `Directionality` and `Text` have no render object of their own; they
   only configure or compose.

The last line is the key to Flutter's performance: pumping a **new** `Greeting` widget produced
the **same** element. Widgets are cheap throwaway descriptions rebuilt constantly; elements and
render objects are long-lived and updated in place.

A detail you'll meet in every test: the text measured 448 × 32. The test environment uses a
special font in which every glyph is a square box the size of the font — 14 characters × 32
logical pixels = 448. Real devices use real fonts, so never assert pixel widths of text in
tests. The default test "screen" is 800 × 600 logical pixels.

## What this design buys and costs

| | Consequence |
|---|---|
| Draws its own pixels | UI looks identical across platforms; no "works on Android, broken on iOS" layout bugs |
| Engine ships in the app | larger downloads than a pure-native app; startup has to load the engine |
| Everything is Dart | one language for UI and logic; full source of the framework is readable |
| Not native controls | platform look-and-feel must be chosen (Material, Cupertino, adaptive widgets); accessibility and text input go through Flutter's own bridges |
| Hot reload | very fast iteration in debug builds |

Flutter fits apps with custom, branded UI that must look the same everywhere, teams that want
one codebase for mobile (and often web/desktop), and UIs heavy on animation. It's a weaker fit
for apps that are mostly thin wrappers around platform-specific features, or that must look
exactly like each platform's latest native controls on day one.

## How It Actually Works

`runApp(widget)` attaches your root widget to the **binding**, the object that connects the
framework to the engine. When the engine signals that a new frame is due (typically 60 or 120
times a second, but only when something has changed), the binding runs a frame:

1. **Build** — dirty elements call their widget's `build` and **reconcile** the result with
   their existing children: if the new child widget has the same type (and key) as the old one,
   the element is kept and updated; otherwise it's replaced. That's why the rebuild above kept the
   same element.
2. **Layout** — render objects that need it lay themselves out, constraints flowing down and
   sizes coming back up ([lesson 04](04-layout-constraints.md)).
3. **Paint** — render objects record drawing commands into layers.
4. **Composite** — the layer tree is handed to the engine, which rasterises it on the GPU.

`flutter test` runs the same framework against a fake binding (`TestWidgetsFlutterBinding`)
with no GPU and no window, which is why widget tests run in milliseconds on a laptop or in CI.
`tester.pumpWidget` runs exactly one frame of the pipeline above. [Level 4 · 01](../level-4/01-rendering-pipeline.md)
goes through each phase in depth.

## Common misconceptions

- **"Flutter compiles to native widgets."** It doesn't; it draws its own. (React Native and
  .NET MAUI take the native-widget approach.)
- **"Rebuilding widgets is expensive, so avoid `build`."** Widgets are small immutable objects;
  creating thousands per frame is normal. What matters is not doing expensive *work* in `build`.
- **"Dart is only for Flutter."** It's a general-purpose language — this course assumes you know
  it from the [Dart Mastery Path](https://sigilipelli.github.io/dart-mastery-path/).
- **"Text widths in tests match the device."** The test font is a box font.

## Exercise

1. Add a second `Text` below the first inside a `Column` (you'll need `import` of nothing new —
   `Column` is in `widgets.dart`). Re-run the tree walk: which new render object appears?
2. Remove `Directionality` and run the test. Read the error message; what does it tell you about
   why `Text` needs it?
3. Change the font size to 20 and predict the new text size before running the test.
4. In the rebuild check, pump `const Center(child: Text('x'))` the second time instead. Is the
   element still identical? Why not?
