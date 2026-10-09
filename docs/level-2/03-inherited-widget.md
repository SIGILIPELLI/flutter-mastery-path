---
description: "Lift state up, then share it without prop drilling: build an InheritedWidget, see exactly which widgets rebuild, and learn why every state library uses it."
---

# 03 · Lifting State & InheritedWidget

Level 1 kept state inside the widget that used it. Real screens break that quickly: a units toggle in a
settings panel must change a distance label three widgets away, on a different branch of the tree. The first
fix is **lifting state up** — move it to the nearest common ancestor and pass values down and callbacks up.
The second problem appears right after: passing the same value through five constructors that don't use it
("prop drilling"). Flutter's built-in answer is `InheritedWidget`, and it's worth learning properly even if
you'll use Provider or Riverpod day to day, because both are built on it.

## Lifting state up, briefly

```dart
class Parent extends StatefulWidget { ... }
class _ParentState extends State<Parent> {
  bool metric = true;
  @override
  Widget build(BuildContext context) => Column(children: [
        DistanceLabel(km: 5, metric: metric),                       // value down
        UnitsSwitch(metric: metric, onChanged: (v) => setState(() => metric = v)), // event up
      ]);
}
```

This is the right first move and is often enough. The pain starts when `metric` has to travel through
`RunScreen → StatsCard → Row → DistanceLabel`, and every one of those constructors grows a parameter it doesn't
care about.

## An InheritedWidget for app settings

Split the job in two: a `StatefulWidget` **owns** the state, and an `InheritedWidget` **delivers** it to any
descendant that asks.

```dart title="inherited_demo.dart"
import 'package:flutter/material.dart';

/// Immutable settings value shared down the tree.
@immutable
class Settings {
  const Settings({required this.metric, required this.fontScale});
  final bool metric;
  final double fontScale;
  Settings copyWith({bool? metric, double? fontScale}) =>
      Settings(metric: metric ?? this.metric, fontScale: fontScale ?? this.fontScale);
}

/// Exposes [Settings] to descendants. Rebuilds dependents only when the value changes.
class SettingsScope extends InheritedWidget {
  const SettingsScope({super.key, required this.settings, required this.update, required super.child});
  final Settings settings;
  final void Function(Settings) update;

  static SettingsScope of(BuildContext context) {
    final scope = context.dependOnInheritedWidgetOfExactType<SettingsScope>();
    assert(scope != null, 'No SettingsScope above this context');
    return scope!;
  }

  @override
  bool updateShouldNotify(SettingsScope old) => settings != old.settings;
}

/// Owns the state; the InheritedWidget is just the delivery mechanism.
class SettingsHost extends StatefulWidget {
  const SettingsHost({super.key, required this.child});
  final Widget child;
  @override
  State<SettingsHost> createState() => _SettingsHostState();
}

class _SettingsHostState extends State<SettingsHost> {
  Settings _settings = const Settings(metric: true, fontScale: 1.0);
  @override
  Widget build(BuildContext context) => SettingsScope(
        settings: _settings,
        update: (s) => setState(() => _settings = s),
        child: widget.child,
      );
}

/// Counts builds so we can see who rebuilt.
final buildLog = <String>[];

class DistanceLabel extends StatelessWidget {
  const DistanceLabel({super.key, required this.km});
  final double km;
  @override
  Widget build(BuildContext context) {
    buildLog.add('DistanceLabel');
    final s = SettingsScope.of(context).settings;
    final text = s.metric ? '${km.toStringAsFixed(1)} km' : '${(km * 0.621371).toStringAsFixed(1)} mi';
    return Text(text, textScaler: TextScaler.linear(s.fontScale));
  }
}

class StaticHeader extends StatelessWidget {
  const StaticHeader({super.key});
  @override
  Widget build(BuildContext context) {
    buildLog.add('StaticHeader');
    return const Text('Today\'s run');
  }
}

class UnitsSwitch extends StatelessWidget {
  const UnitsSwitch({super.key});
  @override
  Widget build(BuildContext context) {
    buildLog.add('UnitsSwitch');
    final scope = SettingsScope.of(context);
    return Switch(
      value: scope.settings.metric,
      onChanged: (v) => scope.update(scope.settings.copyWith(metric: v)),
    );
  }
}

class RunScreen extends StatelessWidget {
  const RunScreen({super.key});
  @override
  Widget build(BuildContext context) => const Column(children: [
        StaticHeader(),
        DistanceLabel(km: 5),
        UnitsSwitch(),
      ]);
}
```

The pieces:

- `SettingsScope.of(context)` calls `context.dependOnInheritedWidgetOfExactType<SettingsScope>()`. That does two
  things: it finds the nearest `SettingsScope` above `context`, and it **registers this widget as a dependent**.
- `updateShouldNotify` decides whether dependents need to rebuild when a new `SettingsScope` replaces the old
  one. Comparing the settings means a rebuild of the host with an *identical* value is free.
- `Settings` is immutable with `copyWith`. Changing settings means creating a new object, which makes
  `settings != old.settings` a meaningful test. (Here `!=` is identity, because `Settings` doesn't override
  `==`; a new object always counts as a change. Override `==`/`hashCode` if you want value comparison.)
- `update` is a callback stored on the inherited widget, so descendants can request changes without knowing
  who owns the state.

## Who rebuilds?

```dart title="inherited_test.dart"
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:l2/s/inherited_demo.dart';

void main() {
  testWidgets('only dependents rebuild when settings change', (tester) async {
    await tester.pumpWidget(const MaterialApp(home: Scaffold(body: SettingsHost(child: RunScreen()))));
    print('first frame built: $buildLog');
    print('label: ${tester.widget<Text>(find.textContaining('km')).data}');

    buildLog.clear();
    await tester.tap(find.byType(Switch));
    await tester.pump();
    print('after toggle rebuilt: $buildLog');
    print('label: ${tester.widget<Text>(find.textContaining('mi')).data}');
  });

  testWidgets('of() without a scope fails loudly', (tester) async {
    await tester.pumpWidget(const MaterialApp(home: DistanceLabel(km: 1)));
    final error = tester.takeException();
    print('error: ${error.runtimeType}: ${error.toString().split('\n').first}');
  });
}
```

```console
$ flutter test test/s/inherited_test.dart
first frame built: [StaticHeader, DistanceLabel, UnitsSwitch]
label: 5.0 km
after toggle rebuilt: [DistanceLabel, UnitsSwitch]
label: 3.1 mi
error: _AssertionError: 'package:l2/s/inherited_demo.dart': Failed assertion: line 21 pos 12: 'scope != null': No SettingsScope above this context
00:00 +2: All tests passed!
```

The important line is `after toggle rebuilt: [DistanceLabel, UnitsSwitch]`. Tapping the switch called
`setState` in `_SettingsHostState`, which rebuilt `SettingsScope` — but **`StaticHeader` did not rebuild**, and
neither did `RunScreen`. Two things made that happen:

1. `RunScreen` is passed into `SettingsHost` as `child` and created with `const`. When the host rebuilds, it hands
   the *same* child instance back to `SettingsScope`, and Flutter skips rebuilding a subtree whose widget is
   identical to last frame's.
2. Only the widgets that called `SettingsScope.of(context)` were registered as dependents, so only they were
   marked dirty.

That's the core trick behind every Flutter state-management library: put the state above a stable `child`, and
let *dependents* subscribe.

The second test shows what happens when someone forgets the scope: the `assert` produces a readable message in
debug builds. (Asserts are stripped in release builds, so a `!` on `null` would crash there instead. Many APIs
offer both `of` and `maybeOf` — the latter returns `null` — for exactly this reason.)

## Built-in examples you already use

`Theme.of(context)`, `MediaQuery.of(context)`, `Directionality.of(context)`, `DefaultTextStyle.of(context)` and
`Navigator.of(context)` all find an ancestor the same way. That's why changing the theme or rotating the device
rebuilds exactly the widgets that read the theme or media query. Newer APIs go further:
`MediaQuery.sizeOf(context)` subscribes only to *size* changes, not to every media-query field, using
`InheritedModel` — an `InheritedWidget` variant whose dependents declare which "aspect" they care about.

## When InheritedWidget isn't enough

Hand-written inherited widgets get repetitive: a stateful host, an inherited scope, an `of` method, a
`copyWith`... and they say nothing about async loading, disposal, or testing overrides. That boilerplate is what
[Provider](04-provider.md) packages up, and [Riverpod](05-riverpod.md) rethinks. Use a raw `InheritedWidget` for
small, stable, app-wide values (a feature-flag set, a theme-like config), or when writing a package that
shouldn't depend on a state library.

## How It Actually Works

Every `Element` (the live object behind a widget, covered in [Level 3 · 01](../level-3/01-element-tree.md))
keeps a map of the inherited elements above it, keyed by type. It's inherited from the parent and extended when
an `InheritedElement` is mounted — so `dependOnInheritedWidgetOfExactType` is a hash lookup, O(1), not a walk up
the tree. The call also adds the caller to the inherited element's set of dependents.

When an `InheritedElement` is updated with a new widget, it calls `updateShouldNotify(oldWidget)`. If that
returns `true`, it calls `didChangeDependencies()` on every dependent element, which marks each one dirty; the
next frame rebuilds them. Non-dependents are untouched. This is also why calling `of(context)` inside
`initState` is an error — the element isn't fully mounted yet; read inherited values in `build` or
`didChangeDependencies` instead.

## Common mistakes

- **Putting the state in the `InheritedWidget`.** Inherited widgets are immutable; state belongs in a
  `State` object above it.
- **Building the subtree inside the host's `build`** (`SettingsScope(child: RunScreen())` without `const` and
  without passing `child` through). Then every settings change rebuilds the whole screen, defeating the point.
- **Mutating the settings object in place** and calling `setState`: `updateShouldNotify` compares the same
  object with itself and nobody rebuilds.
- **Calling `of(context)` with a context *above* the scope** — e.g. the context of the widget that creates
  `SettingsHost`. Use a `Builder` or a separate widget to get a context below it.
- **Using `getInheritedWidgetOfExactType`** (no dependency) when you actually needed to react to changes.

## Exercise

1. Add a font-size slider to the screen that updates `fontScale`. Verify with `buildLog` that `StaticHeader`
   still doesn't rebuild.
2. Override `==` and `hashCode` on `Settings`. Write a test where `update` is called with an equal-but-new
   object and confirm nobody rebuilds.
3. Add `SettingsScope.maybeOf` and use it in a widget that falls back to metric when no scope exists.
4. Convert `SettingsScope` to an `InheritedModel<String>` with aspects `'units'` and `'font'`, so the distance
   label rebuilds for both but a new `FontPreview` widget rebuilds only for `'font'`.
