---
description: "One Flutter app for phones, tablets and desktops: window size classes, NavigationBar vs NavigationRail, list–detail panes and LayoutBuilder."
---

# 04 · Responsive & Adaptive Layouts

A Flutter app can run on a 360-pixel-wide phone, a foldable that changes width while open, a tablet in split-screen, a
desktop window the user resizes freely, and a browser tab. Two related ideas handle this:

- **Responsive**: the layout changes with the *space available* — one column on a phone, list + detail side by side on a
  desktop.
- **Adaptive**: the app adapts to the *platform and input* — mouse hover and keyboard shortcuts on desktop, platform-
  conventional controls and navigation on iOS vs Android.

This lesson builds a mail-style shell that switches navigation style and pane layout at Material 3's window-size
breakpoints, and tests it at phone, tablet and desktop sizes.

## Breakpoints: window size classes

Material 3 defines width classes in logical pixels: **compact** (< 600), **medium** (600–839) and **expanded** (840+),
with larger classes above that for very wide screens. Logical pixels already account for pixel density, so a 390-wide
phone is "compact" regardless of how many physical pixels it has. Pick breakpoints by *content* (when does the list look
silly stretched?), but starting from the standard classes keeps you consistent with platform guidance.

## The adaptive shell

```dart title="responsive_demo.dart"
import 'package:flutter/material.dart';

/// Material 3 window size classes (widths in logical pixels).
enum WindowSize { compact, medium, expanded }

WindowSize windowSizeFor(double width) => width < 600
    ? WindowSize.compact
    : width < 840
        ? WindowSize.medium
        : WindowSize.expanded;

const destinations = [
  (icon: Icons.inbox, label: 'Inbox'),
  (icon: Icons.star, label: 'Starred'),
  (icon: Icons.send, label: 'Sent'),
];

/// Adapts the whole shell to the window: bottom bar, rail, or extended rail + detail pane.
class AdaptiveShell extends StatefulWidget {
  const AdaptiveShell({super.key});
  @override
  State<AdaptiveShell> createState() => _AdaptiveShellState();
}

class _AdaptiveShellState extends State<AdaptiveShell> {
  int index = 0;
  int? openMessage;

  @override
  Widget build(BuildContext context) {
    // sizeOf subscribes only to size changes, not every MediaQuery field.
    final size = windowSizeFor(MediaQuery.sizeOf(context).width);
    final list = MessageList(onOpen: (i) => setState(() => openMessage = i), selected: openMessage);

    final body = switch (size) {
      WindowSize.expanded => Row(children: [
          SizedBox(width: 360, child: list),
          const VerticalDivider(width: 1),
          Expanded(child: openMessage == null ? const Center(child: Text('Select a message')) : MessageDetail(id: openMessage!)),
        ]),
      _ => list,
    };

    if (size == WindowSize.compact) {
      return Scaffold(
        appBar: AppBar(title: Text(destinations[index].label)),
        body: body,
        bottomNavigationBar: NavigationBar(
          selectedIndex: index,
          onDestinationSelected: (i) => setState(() => index = i),
          destinations: [for (final d in destinations) NavigationDestination(icon: Icon(d.icon), label: d.label)],
        ),
      );
    }
    return Scaffold(
      body: Row(children: [
        NavigationRail(
          extended: size == WindowSize.expanded,
          selectedIndex: index,
          onDestinationSelected: (i) => setState(() => index = i),
          labelType: size == WindowSize.medium ? NavigationRailLabelType.all : NavigationRailLabelType.none,
          destinations: [for (final d in destinations) NavigationRailDestination(icon: Icon(d.icon), label: Text(d.label))],
        ),
        const VerticalDivider(width: 1),
        Expanded(child: body),
      ]),
    );
  }
}

class MessageList extends StatelessWidget {
  const MessageList({super.key, required this.onOpen, this.selected});
  final ValueChanged<int> onOpen;
  final int? selected;
  @override
  Widget build(BuildContext context) => ListView.builder(
        itemCount: 20,
        itemBuilder: (context, i) => ListTile(
          selected: i == selected,
          title: Text('Message $i'),
          onTap: () {
            onOpen(i);
            // On narrow layouts there's no detail pane: push a page instead.
            if (windowSizeFor(MediaQuery.sizeOf(context).width) != WindowSize.expanded) {
              Navigator.of(context).push(MaterialPageRoute(builder: (_) => Scaffold(appBar: AppBar(), body: MessageDetail(id: i))));
            }
          },
        ),
      );
}

class MessageDetail extends StatelessWidget {
  const MessageDetail({super.key, required this.id});
  final int id;
  @override
  Widget build(BuildContext context) => Center(child: Text('Body of message $id'));
}

/// LayoutBuilder responds to the space *this widget* gets, not the window.
class StatTiles extends StatelessWidget {
  const StatTiles({super.key});
  @override
  Widget build(BuildContext context) => LayoutBuilder(builder: (context, constraints) {
        final columns = (constraints.maxWidth / 180).floor().clamp(1, 4);
        return GridView.count(
          crossAxisCount: columns,
          shrinkWrap: true,
          children: [for (var i = 0; i < 6; i++) Card(child: Center(child: Text('Stat $i')))],
        );
      });
}

```

Two different questions are answered by two different tools:

- **"How big is the window?"** → `MediaQuery.sizeOf(context)`. Use it for app-level structure (which navigation, how many
  panes). `sizeOf` subscribes only to size changes; the older `MediaQuery.of(context).size` rebuilds on *any* media-query
  change, including the keyboard appearing.
- **"How much space did my parent give me?"** → `LayoutBuilder`. Use it for components: `StatTiles` doesn't care about
  the window, only its own width, so it works the same in a narrow side panel on a desktop and full-width on a phone.

## Testing at three sizes

```dart title="responsive_test.dart"
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:l2/a/responsive_demo.dart';

void main() {
  Future<void> at(WidgetTester tester, double width, double height) async {
    tester.view.physicalSize = Size(width, height);
    tester.view.devicePixelRatio = 1;
    addTearDown(tester.view.reset);
    await tester.pumpWidget(const SizedBox()); // start from a fresh app each time
    await tester.pumpWidget(const MaterialApp(home: AdaptiveShell()));
  }

  String describe() => [
        if (find.byType(NavigationBar).evaluate().isNotEmpty) 'bottom bar',
        if (find.byType(NavigationRail).evaluate().isNotEmpty)
          'rail(extended=${(find.byType(NavigationRail).evaluate().single.widget as NavigationRail).extended})',
        if (find.text('Select a message').evaluate().isNotEmpty) 'detail pane',
      ].join(' + ');

  for (final (w, h) in [(390.0, 844.0), (700.0, 1000.0), (1280.0, 800.0)]) {
    testWidgets('layout at ${w.toInt()}x${h.toInt()}', (tester) async {
      await at(tester, w, h);
      print('${w.toInt().toString().padLeft(4)} wide -> ${windowSizeFor(w).name.padRight(8)} ${describe()}');
    });
  }

  testWidgets('tap a message: push on phone, pane on desktop', (tester) async {
    for (final w in [390.0, 1280.0]) {
      await at(tester, w, 800);
      await tester.tap(find.text('Message 2'));
      await tester.pumpAndSettle();
      final routes = find.byType(BackButton).evaluate().length;
      print('${w.toInt()}: detail shown=${find.text('Body of message 2').evaluate().length == 1}, back button=${routes == 1}');
    }
  });

  testWidgets('selection survives resizing', (tester) async {
    await at(tester, 1280, 800);
    await tester.tap(find.text('Message 5'));
    await tester.pump();
    tester.view.physicalSize = const Size(700, 800);
    await tester.pump();
    tester.view.physicalSize = const Size(1280, 800);
    await tester.pump();
    print('after shrinking and growing: ${find.text('Body of message 5').evaluate().length == 1 ? 'still showing message 5' : 'lost'}');
  });

  testWidgets('LayoutBuilder columns follow the available width', (tester) async {
    final out = <String>[];
    for (final w in [150.0, 400.0, 760.0, 2000.0]) {
      await tester.pumpWidget(MaterialApp(home: Align(alignment: Alignment.topLeft, child: SizedBox(width: w, child: const StatTiles()))));
      final grid = tester.widget<GridView>(find.byType(GridView));
      out.add('${w.toInt()}px->${(grid.childrenDelegate as SliverChildListDelegate).children.length} tiles in ${(grid.gridDelegate as SliverGridDelegateWithFixedCrossAxisCount).crossAxisCount} cols');
    }
    print(out.join(', '));
  });

}
```

```console
$ flutter test test/a/responsive_test.dart
 390 wide -> compact  bottom bar
 700 wide -> medium   rail(extended=false)
1280 wide -> expanded rail(extended=true) + detail pane
390: detail shown=true, back button=true
1280: detail shown=true, back button=false
after shrinking and growing: still showing message 5
150px->6 tiles in 1 cols, 400px->6 tiles in 2 cols, 760px->6 tiles in 4 cols, 2000px->6 tiles in 4 cols
00:00 +6: All tests passed!
```

- **Three shells from one widget**: a bottom `NavigationBar` on the phone, a compact `NavigationRail` with labels on the
  tablet, an extended rail plus a detail pane on the desktop.
- **Same tap, different behaviour**: on the phone, opening a message pushed a page (with a back button); on the desktop it
  filled the pane and nothing was pushed.
- **State survives resizing**: the selected message is stored in `_AdaptiveShellState`, *above* the layout switch, so
  shrinking the window to tablet width and back didn't lose it. If the selection lived in the detail pane's own state, the
  pane would be unmounted when the layout changed (the element-tree rule from [lesson 01](01-element-tree.md)) and the
  selection lost.
- **`LayoutBuilder` columns** followed the given width — and capped at 4 so tiles don't become tiny on huge screens.

`tester.view.physicalSize` and `devicePixelRatio` set the virtual screen; resetting them in `addTearDown` keeps other tests
unaffected.

## Adaptive details worth handling

- **Navigation conventions**: `NavigationBar` for compact, `NavigationRail` for medium/expanded, `NavigationDrawer` when
  there are many destinations.
- **Input**: desktop and web users have a mouse and keyboard. Give interactive elements hover states (Material widgets do),
  add keyboard shortcuts with `Shortcuts`/`Actions` or `CallbackShortcuts`, and make sure focus traversal with Tab works.
  Don't hide functionality behind long-press only.
- **Platform look**: `Switch.adaptive`, `Slider.adaptive`, `CircularProgressIndicator.adaptive` and
  `AlertDialog.adaptive` render Cupertino-style on Apple platforms and Material elsewhere. Most apps use their own brand
  design everywhere and adapt only where platform expectations are strong (dialogs, pickers, back gestures).
- **Text scale**: users can enlarge text. Test layouts at large text scales with `MediaQuery(data: ...copyWith(textScaler:
  TextScaler.linear(2)))` — overflow at 200 % text is a real accessibility bug ([lesson 08](08-a11y-i18n.md)).
- **Safe areas and cut-outs**: `SafeArea` (or `Scaffold`, which handles most cases) keeps content out of notches and system
  bars.
- **Orientation**: prefer width-based decisions over `Orientation`; a landscape phone is still narrower than a portrait
  tablet.

## How It Actually Works

The engine reports each view's physical size, device pixel ratio, padding (system bars), view insets (keyboard) and
platform settings to the framework. `WidgetsBinding` turns that into `MediaQueryData` and places a `MediaQuery` widget near
the root (inside `MaterialApp`'s `View`). When the window resizes, the binding gets a metrics-changed callback, builds new
`MediaQueryData`, and dependents rebuild. `MediaQuery` is an `InheritedModel`, which is how `sizeOf` can depend only on the
`size` aspect.

`LayoutBuilder` is different: it defers building its child until **layout**, when its own constraints are known. Its render
object calls your builder during `performLayout` with the actual `BoxConstraints`, then lays out whatever you returned.
That's why it can react to the space a parent gives, but also why you can't use its result to change the parent's size —
constraints only go down ([Level 1 · 04](../level-1/04-layout-constraints.md)).

## Common mistakes

- **Branching on `Platform.isAndroid`/`isIOS` for layout.** Platform isn't size: iPads, Android tablets, desktops and the
  web break that logic. Branch on width; use platform only for platform conventions. (Also, `dart:io`'s `Platform` throws
  on the web; `defaultTargetPlatform` works everywhere.)
- **`MediaQuery.of(context).size` inside deep components** — they should use `LayoutBuilder`, and the `of` form causes
  extra rebuilds.
- **Keeping UI state inside a pane that's removed at some sizes**, so resizing resets it.
- **Hard-coded widths** (`width: 375`) that only fit one device.
- **Testing only on your own phone.** Use the size loop above, and resize a desktop/web build by hand.

## Exercise

1. Add a "large" class (≥ 1200) that shows a third column with message metadata. Test at 1400 wide.
2. On medium width, show the detail as a side sheet over the list instead of pushing a page.
3. Add `Ctrl+N`/`⌘N` for "new message" using `CallbackShortcuts`, and a test that sends the key event with
   `tester.sendKeyDownEvent`.
4. Run the 390-wide test with `TextScaler.linear(2.0)`. Does anything overflow? Fix it.
