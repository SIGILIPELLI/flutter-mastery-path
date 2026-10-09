---
description: "Level 1 project: a Flutter tip splitter with integer-cent money logic, live recalculation, Material 3 controls and 12 unit and widget tests."
---

# 10 · Project — A Tip Splitter with Widget Tests

Level 1's project is small on purpose so every part of it can be done properly: a tip calculator
that splits a bill between friends. It uses everything from this level — composition, layout,
state, text input, theming — and it's backed by a test suite that covers the arithmetic, every
control, invalid input, the edge cases of rounding, and a dark-mode, small-phone render. Along the
way the tests caught two real mistakes, both shown below.

## Requirements

- Type a bill amount; results update as you type. Accept `84.5`, `84,50` and surrounding spaces.
- Choose a 10, 15 or 20 % tip; choose the number of people (1–50).
- Show tip, total, and each person's share. Shares are never rounded *down* — the group must cover the
  bill — and any extra cents are shown.
- Optional "round each share up" to the next whole unit.
- Clear error for invalid amounts; no results until the amount is valid.
- Works in light and dark mode on a small phone.

## Structure

```text
tip_splitter/
├── lib/
│   ├── main.dart            app + theme
│   └── src/
│       ├── split.dart       pure Dart: money logic, parsing, formatting
│       └── tip_screen.dart  the UI
└── test/
    ├── split_test.dart      unit tests
    └── tip_screen_test.dart widget tests
```

The rule that shapes everything: **logic goes in plain Dart, UI goes in widgets**. `split.dart` doesn't
import Flutter at all, so its tests are microseconds-fast and its behaviour is easy to reason about.

## The money logic

```dart title="split.dart"
/// Pure calculation logic — no Flutter imports, so it's trivially unit-testable.
/// Money is handled in integer cents to avoid floating-point rounding errors.
class BillSplit {
  const BillSplit({required this.tipCents, required this.totalCents, required this.perPersonCents, required this.leftoverCents});
  final int tipCents;
  final int totalCents;
  final int perPersonCents;

  /// Cents collected beyond the total when everyone pays [perPersonCents].
  final int leftoverCents;
}

BillSplit calculateSplit({required int billCents, required int tipPercent, required int people, bool roundUp = false}) {
  if (billCents < 0) throw ArgumentError.value(billCents, 'billCents', 'must not be negative');
  if (people < 1) throw ArgumentError.value(people, 'people', 'must be at least 1');
  if (tipPercent < 0 || tipPercent > 100) throw ArgumentError.value(tipPercent, 'tipPercent', 'must be 0..100');

  final tip = (billCents * tipPercent / 100).round();
  final total = billCents + tip;
  // Ceil division so the group never pays less than the total.
  var perPerson = (total + people - 1) ~/ people;
  if (roundUp) perPerson = ((perPerson + 99) ~/ 100) * 100; // to the next whole unit
  return BillSplit(tipCents: tip, totalCents: total, perPersonCents: perPerson, leftoverCents: perPerson * people - total);
}

/// Parses user input like "84.5", "84,50" or "  84 " into cents. Returns null if invalid.
int? parseCents(String input) {
  final s = input.trim().replaceAll(',', '.');
  if (!RegExp(r'^\d{1,7}(\.\d{0,2})?$').hasMatch(s)) return null;
  final parts = s.split('.');
  final whole = int.parse(parts[0]);
  final frac = parts.length > 1 ? parts[1].padRight(2, '0') : '00';
  return whole * 100 + int.parse(frac.isEmpty ? '00' : frac);
}

String formatCents(int cents) {
  final whole = cents ~/ 100;
  final frac = (cents % 100).toString().padLeft(2, '0');
  final grouped = whole.toString().replaceAllMapped(RegExp(r'\B(?=(\d{3})+(?!\d))'), (_) => ',');
  return '$grouped.$frac';
}
```

Decisions worth defending:

- **Integer cents, never `double`.** `0.1 + 0.2` is not `0.3` in floating point; with cents, `8450 * 15 /
  100` is computed once and rounded once.
- **Ceiling division for shares.** `(total + people - 1) ~/ people` rounds up, so three people splitting
  100.00 pay 33.34 each and the 0.02 surplus is reported, instead of collectively paying 99.99.
- **`parseCents` is strict about format** (at most two decimal places, digits only) and tolerant of
  harmless variation (comma decimal separator, spaces). It returns `null` rather than throwing, which is
  what the UI needs.
- A real app serving several countries would use the `intl` package for locale-aware number formatting
  ([Level 3 · 08](../level-3/08-a11y-i18n.md)); `formatCents` keeps this project dependency-free.

### A name collision

The class was first called `Split`. The widget code failed to compile:

```text
error • The name 'Split' is defined in the libraries 'package:flutter/src/animation/curves.dart (via package:flutter/material.dart)' and 'package:tip_splitter/src/split.dart'. Try using 'as prefix' for one of the import directives, or hiding the name from all but one of the imports • lib/src/tip_screen.dart:100:9 • ambiguous_import
```

`package:flutter/material.dart` exports a lot of names, including an animation curve called `Split`.
Renaming to `BillSplit` was the clean fix; `import 'package:flutter/material.dart' hide Split;` would also
work. Domain-specific names collide less.

## The screen

```dart title="tip_screen.dart"
import 'package:flutter/material.dart';
import 'split.dart';

class TipScreen extends StatefulWidget {
  const TipScreen({super.key});
  @override
  State<TipScreen> createState() => _TipScreenState();
}

class _TipScreenState extends State<TipScreen> {
  final _bill = TextEditingController();
  int _tipPercent = 15;
  int _people = 2;
  bool _roundUp = false;

  @override
  void initState() {
    super.initState();
    _bill.addListener(() => setState(() {})); // recompute as the user types
  }

  @override
  void dispose() {
    _bill.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final cents = parseCents(_bill.text);
    final invalid = _bill.text.isNotEmpty && cents == null;
    final split = cents == null
        ? null
        : calculateSplit(billCents: cents, tipPercent: _tipPercent, people: _people, roundUp: _roundUp);
    final theme = Theme.of(context);

    return Scaffold(
      appBar: AppBar(title: const Text('Tip Splitter')),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          TextField(
            key: const Key('bill'),
            controller: _bill,
            keyboardType: const TextInputType.numberWithOptions(decimal: true),
            decoration: InputDecoration(
              labelText: 'Bill amount',
              errorText: invalid ? 'Enter an amount like 84.50' : null,
              border: const OutlineInputBorder(),
            ),
          ),
          const SizedBox(height: 16),
          Text('Tip', style: theme.textTheme.titleMedium),
          const SizedBox(height: 8),
          SegmentedButton<int>(
            segments: const [
              ButtonSegment(value: 10, label: Text('10%')),
              ButtonSegment(value: 15, label: Text('15%')),
              ButtonSegment(value: 20, label: Text('20%')),
            ],
            selected: {_tipPercent},
            onSelectionChanged: (s) => setState(() => _tipPercent = s.first),
          ),
          const SizedBox(height: 16),
          Row(
            children: [
              Expanded(child: Text('People', style: theme.textTheme.titleMedium)),
              IconButton(
                tooltip: 'Fewer people',
                onPressed: _people > 1 ? () => setState(() => _people--) : null,
                icon: const Icon(Icons.remove),
              ),
              Text('$_people', key: const Key('people'), style: theme.textTheme.titleLarge),
              IconButton(
                tooltip: 'More people',
                onPressed: _people < 50 ? () => setState(() => _people++) : null,
                icon: const Icon(Icons.add),
              ),
            ],
          ),
          SwitchListTile(
            contentPadding: EdgeInsets.zero,
            title: const Text('Round each share up'),
            value: _roundUp,
            onChanged: (v) => setState(() => _roundUp = v),
          ),
          const Divider(height: 32),
          if (split == null)
            const Text('Enter the bill to see the split.')
          else
            _Results(split: split, people: _people),
        ],
      ),
    );
  }
}

class _Results extends StatelessWidget {
  const _Results({required this.split, required this.people});
  final BillSplit split;
  final int people;

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    Widget row(String label, String value, {Key? key}) => Padding(
          padding: const EdgeInsets.symmetric(vertical: 4),
          child: Row(children: [Expanded(child: Text(label)), Text(value, key: key)]),
        );
    return Column(
      crossAxisAlignment: CrossAxisAlignment.stretch,
      children: [
        row('Tip', formatCents(split.tipCents), key: const Key('tip')),
        row('Total', formatCents(split.totalCents), key: const Key('total')),
        const SizedBox(height: 8),
        Text('Each of $people pays', style: theme.textTheme.titleMedium),
        Text(formatCents(split.perPersonCents), key: const Key('each'), style: theme.textTheme.displaySmall),
        if (split.leftoverCents > 0)
          Text('${formatCents(split.leftoverCents)} extra goes to the tip jar', key: const Key('leftover')),
      ],
    );
  }
}
```

```dart title="main.dart"
import 'package:flutter/material.dart';
import 'src/tip_screen.dart';

void main() => runApp(const TipApp());

class TipApp extends StatelessWidget {
  const TipApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Tip Splitter',
      theme: ThemeData(colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF0B6E4F))),
      darkTheme: ThemeData(colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF0B6E4F), brightness: Brightness.dark)),
      home: const TipScreen(),
    );
  }
}
```

How the UI follows Level 1's lessons:

- **One source of truth.** The controller's text, `_tipPercent`, `_people` and `_roundUp` are the only state;
  the split is *derived* in `build` every time. There's no stored "result" that could get out of sync.
- **Live updates** come from a controller listener that calls `setState`. The listener is added in
  `initState` and the controller disposed in `dispose` ([lesson 05](05-stateful-widgets.md)).
- **`errorText` instead of a `Form`**: there's one field and no submit button, so the error is computed
  directly from the text ([lesson 06](06-forms-input.md)).
- **Disabled, not hidden.** The minus button's `onPressed` is `null` at one person, which greys it out and
  tells assistive technology it's disabled.
- **`ListView` as the body** so the screen scrolls on small phones or with large accessibility fonts,
  instead of overflowing ([lesson 04](04-layout-constraints.md)).
- **Keys on result texts** (`Key('each')`) give tests stable handles that don't depend on wording or layout.
- **A private `_Results` widget** keeps the screen's `build` readable ([lesson 03](03-widgets-composition.md)).

## The tests

```dart title="split_test.dart"
import 'package:flutter_test/flutter_test.dart';
import 'package:tip_splitter/src/split.dart';

void main() {
  group('calculateSplit', () {
    test('basic split', () {
      final s = calculateSplit(billCents: 8450, tipPercent: 15, people: 2);
      expect([s.tipCents, s.totalCents, s.perPersonCents, s.leftoverCents], [1268, 9718, 4859, 0]);
    });
    test('uneven totals round shares up, never down', () {
      final s = calculateSplit(billCents: 10000, tipPercent: 0, people: 3);
      expect([s.perPersonCents, s.leftoverCents], [3334, 2]); // 3 x 33.34 = 100.02
    });
    test('round up to whole units', () {
      final s = calculateSplit(billCents: 8450, tipPercent: 15, people: 2, roundUp: true);
      expect([s.perPersonCents, s.leftoverCents], [4900, 82]);
    });
    test('rejects nonsense', () {
      expect(() => calculateSplit(billCents: 100, tipPercent: 15, people: 0), throwsArgumentError);
      expect(() => calculateSplit(billCents: -1, tipPercent: 15, people: 1), throwsArgumentError);
    });
  });

  group('parseCents', () {
    test('accepts common inputs', () {
      expect(['84.5', '84,50', ' 84 ', '0.07', '84.'].map(parseCents), [8450, 8450, 8400, 7, 8400]);
    });
    test('rejects bad input', () {
      expect(['', 'abc', '1.234', '-5', '1e3', '12345678'].map(parseCents), everyElement(isNull));
    });
  });

  test('formatCents groups thousands', () {
    expect([formatCents(7), formatCents(9718), formatCents(123456789)], ['0.07', '97.18', '1,234,567.89']);
  });
}
```

```dart title="tip_screen_test.dart"
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:tip_splitter/main.dart';

String textOf(WidgetTester t, String key) => t.widget<Text>(find.byKey(Key(key))).data!;

void main() {
  testWidgets('starts with a prompt, then splits as you type', (tester) async {
    await tester.pumpWidget(const TipApp());
    expect(find.text('Enter the bill to see the split.'), findsOneWidget);

    await tester.enterText(find.byKey(const Key('bill')), '84.50');
    await tester.pump();
    expect(textOf(tester, 'tip'), '12.68');
    expect(textOf(tester, 'total'), '97.18');
    expect(textOf(tester, 'each'), '48.59');
    expect(find.byKey(const Key('leftover')), findsNothing);
  });

  testWidgets('changing tip, people and rounding updates the result', (tester) async {
    await tester.pumpWidget(const TipApp());
    await tester.enterText(find.byKey(const Key('bill')), '100');
    await tester.tap(find.text('20%'));
    await tester.tap(find.byTooltip('More people'));
    await tester.pump();
    expect(textOf(tester, 'people'), '3');
    expect(textOf(tester, 'each'), '40.00');

    await tester.tap(find.text('10%'));
    await tester.pump();
    expect(textOf(tester, 'each'), '36.67');
    expect(textOf(tester, 'leftover'), '0.01 extra goes to the tip jar');

    await tester.tap(find.text('Round each share up'));
    await tester.pump();
    expect(textOf(tester, 'each'), '37.00');
  });

  testWidgets('invalid input shows an error and no result', (tester) async {
    await tester.pumpWidget(const TipApp());
    await tester.enterText(find.byKey(const Key('bill')), '12.345');
    await tester.pump();
    expect(find.text('Enter an amount like 84.50'), findsOneWidget);
    expect(find.byKey(const Key('each')), findsNothing);
  });

  testWidgets('people cannot go below one', (tester) async {
    await tester.pumpWidget(const TipApp());
    await tester.tap(find.byTooltip('Fewer people'));
    await tester.pump();
    expect(textOf(tester, 'people'), '1');
    final minus = tester.widget<IconButton>(find.widgetWithIcon(IconButton, Icons.remove));
    expect(minus.onPressed, isNull, reason: 'button disabled at 1');
  });

  testWidgets('renders in dark mode without overflow', (tester) async {
    tester.platformDispatcher.platformBrightnessTestValue = Brightness.dark;
    addTearDown(tester.platformDispatcher.clearPlatformBrightnessTestValue);
    tester.view.physicalSize = const Size(360 * 3, 640 * 3); // a small phone at 3x
    tester.view.devicePixelRatio = 3;
    addTearDown(tester.view.reset);
    await tester.pumpWidget(const TipApp());
    await tester.enterText(find.byKey(const Key('bill')), '1234567.89');
    await tester.pump();
    expect(tester.takeException(), isNull);
    expect(Theme.of(tester.element(find.byKey(const Key('each')))).brightness, Brightness.dark);
    expect(textOf(tester, 'each'), '709,876.54');
  });
}
```

```console
$ flutter test --reporter expanded
00:00 +0: split_test.dart: calculateSplit basic split
00:00 +1: split_test.dart: calculateSplit uneven totals round shares up, never down
00:00 +2: split_test.dart: calculateSplit round up to whole units
00:00 +3: split_test.dart: calculateSplit rejects nonsense
00:00 +4: split_test.dart: parseCents accepts common inputs
00:00 +5: split_test.dart: parseCents rejects bad input
00:00 +6: split_test.dart: formatCents groups thousands
00:00 +7: tip_screen_test.dart: starts with a prompt, then splits as you type
00:00 +8: tip_screen_test.dart: changing tip, people and rounding updates the result
00:00 +9: tip_screen_test.dart: invalid input shows an error and no result
00:00 +10: tip_screen_test.dart: people cannot go below one
00:01 +11: tip_screen_test.dart: renders in dark mode without overflow
00:01 +12: All tests passed!
```

(Paths shortened.) The last widget test sets the platform brightness to dark and the test view to a
360 × 640 phone at 3x, enters a seven-digit bill, and asserts there's no layout exception and that the theme
really is dark. `tester.takeException()` returning `null` is how you assert "no overflow" in a test.

### The test that caught the tester

That last test first failed:

```text
Expected: '711,284.84'
  Actual: '709,876.54'
```

The expected value was computed by hand and was wrong: 1,234,567.89 × 1.15 = 1,419,753.07, and half of that
is 709,876.54 — what the app showed. A failing test means *something* disagrees; it's worth checking the
expectation before "fixing" the code. Unit tests on the pure logic make that check quick.

### Running it for real

```console
$ flutter build web --release
Compiling lib/main.dart for the Web...                             17.7s
✓ Built build/web
```

Mobile builds weren't run on this machine (see [lesson 02](02-setup-first-app.md)); the widget tests exercise
the same Dart code that would run there.

## How It Actually Works

Each keystroke updates the `TextEditingController`'s value, which notifies its listeners; our listener calls
`setState`, marking `_TipScreenState` dirty. On the next frame `build` re-parses the text, recomputes the split
(a few integer operations), and returns a new widget tree. Element reconciliation updates only what changed —
the result texts' `RenderParagraph`s get new strings and are re-laid out and repainted; the segmented button,
the switch and the field itself are updated in place with identical configuration. In tests, `enterText` drives
the same path through `TestTextInput`, and `pump()` runs exactly that one frame, which is why the assertions see
the new numbers immediately.

## Review checklist

- [ ] No `double` anywhere in money calculations.
- [ ] Derived values are computed, not stored.
- [ ] Every control is exercised by at least one widget test.
- [ ] Edge cases (1 person, uneven split, invalid input, very large numbers) are tested.
- [ ] The screen scrolls instead of overflowing on a small phone.

## Exercise

1. Add a "Custom" tip segment that reveals a `Slider` (0–30 %). Test that moving the slider updates the share.
2. Persist the last-used tip percentage across app restarts — you'll have the tools in
   [Level 2 · 08](../level-2/08-persistence.md); for now, design where that code would live.
3. Add a golden test of the results card in light and dark mode (preview of [Level 2 · 09](../level-2/09-testing.md)).
4. Support uneven splits: one person pays for an extra item. Add it to `calculateSplit` first, with unit tests,
   before touching the UI.
