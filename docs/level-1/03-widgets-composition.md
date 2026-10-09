---
description: "Build UIs by composing small widgets: StatelessWidget, constructor parameters, const, and a measured comparison of helper methods versus extracted widgets."
---

# 03 · Widgets, build() & Composition

In Flutter you rarely draw anything. You **compose**: a profile card is a `Card` containing
`Padding` containing a `Row` of an avatar, some text and an icon. Each of those is a widget, and
your own widgets are just more of the same. This lesson builds a small component hierarchy,
then measures — with a build counter — what actually rebuilds when something changes, which
turns the advice "use `const`" and "prefer widgets over helper methods" from folklore into
something you've seen.

## A widget is a class with a `build` method

```dart
class Avatar extends StatelessWidget {
  const Avatar({super.key, required this.initials, required this.online});
  final String initials;
  final bool online;

  @override
  Widget build(BuildContext context) => /* ... */;
}
```

Every widget follows the same pattern:

- **Configuration in final fields**, passed through the constructor. Widgets are immutable; to
  "change" one, the parent builds a new one with different arguments.
- **A `const` constructor** when all fields are final — it costs nothing and enables the
  optimisation measured below.
- **`super.key`** forwarded to the base class. Keys identify widgets across rebuilds
  ([lesson 07](07-lists-keys.md)).
- **`build(BuildContext context)`** returns the widget subtree. It must be fast and free of side
  effects: Flutter may call it many times per second.

## The component hierarchy

```dart title="composition.dart"
import 'package:flutter/material.dart';

/// Counts how many times each widget's build ran, for the test.
final buildCounts = <String, int>{};
void _count(String name) => buildCounts[name] = (buildCounts[name] ?? 0) + 1;

class ProfileCard extends StatelessWidget {
  const ProfileCard({super.key, required this.name, required this.role, this.online = false});

  final String name;
  final String role;
  final bool online;

  @override
  Widget build(BuildContext context) {
    _count('ProfileCard');
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Row(
          children: [
            Avatar(initials: name.substring(0, 1), online: online),
            const SizedBox(width: 12),
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(name, style: Theme.of(context).textTheme.titleMedium),
                  Text(role),
                ],
              ),
            ),
            const CardMenuIcon(), // const: identical every time
          ],
        ),
      ),
    );
  }
}

class Avatar extends StatelessWidget {
  const Avatar({super.key, required this.initials, required this.online});
  final String initials;
  final bool online;

  @override
  Widget build(BuildContext context) {
    _count('Avatar');
    return Badge(
      isLabelVisible: online,
      smallSize: 10,
      child: CircleAvatar(child: Text(initials)),
    );
  }
}

class CardMenuIcon extends StatelessWidget {
  const CardMenuIcon({super.key});

  @override
  Widget build(BuildContext context) {
    _count('CardMenuIcon');
    return const Icon(Icons.more_vert);
  }
}

/// A parent that rebuilds when a button is pressed.
class Team extends StatefulWidget {
  const Team({super.key});
  @override
  State<Team> createState() => _TeamState();
}

class _TeamState extends State<Team> {
  bool _adaOnline = false;

  // A helper *method*: runs as part of _TeamState.build, can never be skipped.
  Widget _header() {
    _count('_header method');
    return const Text('Team');
  }

  @override
  Widget build(BuildContext context) {
    _count('Team');
    return Column(
      children: [
        _header(),
        ProfileCard(name: 'Ada', role: 'Engineer', online: _adaOnline),
        const ProfileCard(name: 'Grace', role: 'Admiral'),
        TextButton(
          onPressed: () => setState(() => _adaOnline = !_adaOnline),
          child: const Text('Toggle Ada'),
        ),
      ],
    );
  }
}
```

`ProfileCard` composes `Card`, `Padding`, `Row`, `Expanded`, `Column`, `Text` and two of our own
widgets. `Team` is a stateful parent (stateful widgets are [lesson 05](05-stateful-widgets.md)) that
toggles whether Ada is online. A global `buildCounts` map records each `build` call — a debugging
trick for this lesson, not production code.

Some framework widgets worth recognising from the code:

| Widget | Job |
|---|---|
| `Card` | Material surface with elevation and rounded corners |
| `Padding` | Insets its child |
| `Row` / `Column` | Lay children out horizontally / vertically |
| `Expanded` | Makes a `Row`/`Column` child take the remaining space |
| `SizedBox` | A fixed-size box — here, a 12-pixel gap |
| `CircleAvatar`, `Badge`, `Icon` | Material visuals |

## What rebuilds?

```dart title="composition_test.dart"
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hello/l1/composition.dart';

void main() {
  testWidgets('which builds run when the parent rebuilds', (tester) async {
    await tester.pumpWidget(const MaterialApp(home: Scaffold(body: Team())));
    print('first frame:  $buildCounts');

    buildCounts.clear();
    await tester.tap(find.text('Toggle Ada'));
    await tester.pump();
    print('after toggle: $buildCounts');

    expect(find.text('Ada'), findsOneWidget);
    expect(find.byType(ProfileCard), findsNWidgets(2));
  });
}
```

```console
$ flutter test test/l1/composition_test.dart
first frame:  {Team: 1, _header method: 1, ProfileCard: 2, Avatar: 2, CardMenuIcon: 2}
after toggle: {Team: 1, _header method: 1, ProfileCard: 1, Avatar: 1}
00:00 +1: All tests passed!
```

On the first frame everything builds once per instance — two cards, two avatars, two menu icons.
After tapping "Toggle Ada", `Team` rebuilt, and then:

- **Ada's `ProfileCard` and `Avatar` rebuilt** — they're created with `online: _adaOnline`, a value
  that changed, so Flutter has a new widget to apply.
- **Grace's `ProfileCard` did not rebuild.** It's written `const ProfileCard(...)`. A `const`
  expression evaluates to the *same object* every time, so when Flutter compares the old widget to
  the new one and finds them **identical**, it skips that whole subtree.
- **No `CardMenuIcon` rebuilt**, even inside Ada's rebuilt card, for the same reason: `const
  CardMenuIcon()` is the same instance on every build.
- **`_header` ran again.** It's a *method* on `_TeamState`, so it's just part of `Team.build`; it can
  never be skipped independently. Had it been a `const Header()` widget, it would have been skipped
  like the menu icon.

This is the measurable reason behind two pieces of standard Flutter advice: **use `const`
wherever you can**, and **prefer small widget classes over helper methods that return widgets**.
Widget classes also get their own `BuildContext`, can be `const`, and show up by name in the
inspector and in tests (`find.byType(Avatar)`).

Don't over-apply it, though. Rebuilding a few `Text` and `Padding` widgets costs microseconds;
Flutter is designed for it. The point of `const` and extraction is to keep rebuilds *proportional
to what changed* as screens grow, not to eliminate every `build` call.

## Composition over configuration

Flutter's own widgets are small and single-purpose: `Padding` only pads, `Center` only centres,
`Opacity` only fades. Instead of a `Container` with twelve optional parameters, you nest focused
widgets. Your own widgets should follow suit:

- **Pass data in, not widgets out.** `ProfileCard(name:, role:, online:)` is easier to use and test
  than one that takes pre-built `Text` widgets — unless the caller really needs control, in which
  case a `Widget child` / `Widget? trailing` parameter (the "slot" pattern `Scaffold` and `ListTile`
  use) is the idiom.
- **Keep `build` declarative.** It reads data and returns widgets. Fetching data, starting timers or
  writing to storage in `build` causes repeated work and bugs.
- **Name things by role.** `Avatar` and `CardMenuIcon` say what they are; `_buildLeftColumn()` doesn't.

## How It Actually Works

When `setState` marks `_TeamState` dirty, the next frame calls `Team.build` and gets a new list of
child widgets. Each child **element** then runs `updateChild(oldChild, newWidget)`:

1. If `newWidget` is **identical** (same object) to the element's current widget, return immediately —
   nothing below is visited. This is the `const` fast path.
2. Otherwise, if `Widget.canUpdate(old, new)` is true (same `runtimeType` and same `key`), keep the
   element, store the new widget in it, and — for a stateless widget — call `build` again to update
   its children recursively.
3. Otherwise, unmount the old element and inflate a new one from the new widget.

Ada's `ProfileCard` took path 2 (new object, same type); Grace's took path 1. A helper method has no
element of its own, so there's no step at which it could be skipped. `const` canonicalisation is a
Dart compile-time feature: two `const` expressions with the same arguments produce the same object,
which is what makes the identity check possible.

## Common mistakes

- **Doing work in `build`** — network calls, parsing, sorting large lists on every frame.
- **Forgetting `const`** on widgets with no changing input (icons, gaps, static text).
- **Giant `build` methods** with many `_buildX()` helpers; extract widgets instead.
- **Mutable fields in a `StatelessWidget`.** They won't trigger rebuilds and the analyzer warns
  (`@immutable`). Use a `StatefulWidget` or external state.
- **Premature micro-optimisation** — splitting every `Text` into its own class. Extract for clarity
  first; performance usually follows.

## Exercise

1. Turn `_header()` into a `const Header()` widget and re-run the test. What changes in the counts?
2. Give `ProfileCard` an optional `Widget? trailing` slot that defaults to the menu icon. Use it to
   show a `Switch` on Ada's card.
3. Remove `const` from Grace's card and predict the counts after the toggle before running the test.
4. Add a `find.byType(Avatar)` assertion that checks both avatars exist, and a `find.descendant`
   query that finds the `Text('A')` inside Ada's avatar only.
