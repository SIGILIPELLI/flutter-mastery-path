---
description: "Material 3 theming in Flutter: ColorScheme.fromSeed for light and dark, component themes, Theme.of, and measured contrast ratios."
---

# 08 · Material 3 Theming

Hard-coding colours and text styles in every widget works for one screen and fails at ten — and
fails completely the day someone asks for dark mode. Flutter centralises visual decisions in a
`ThemeData` that every Material widget reads from. With **Material 3** (the default in current
Flutter), you can derive an entire accessible colour scheme for light and dark modes from a single
brand colour. This lesson builds a theme, then *measures* what Flutter generated: actual colour
values, contrast ratios, and what the widgets picked up.

## A theme in one function

```dart title="theme.dart"
import 'package:flutter/material.dart';

const brandSeed = Color(0xFF0B6E4F); // a deep green brand colour

ThemeData buildTheme(Brightness brightness) {
  final scheme = ColorScheme.fromSeed(seedColor: brandSeed, brightness: brightness);
  return ThemeData(
    colorScheme: scheme,
    // Component themes: change every widget of a kind in one place.
    filledButtonTheme: FilledButtonThemeData(
      style: FilledButton.styleFrom(
        minimumSize: const Size.fromHeight(48),
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
      ),
    ),
    inputDecorationTheme: const InputDecorationTheme(border: OutlineInputBorder()),
    textTheme: const TextTheme(
      headlineMedium: TextStyle(fontWeight: FontWeight.w700, letterSpacing: -0.5),
    ),
  );
}

class ThemedApp extends StatelessWidget {
  const ThemedApp({super.key, this.mode = ThemeMode.system});
  final ThemeMode mode;

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      theme: buildTheme(Brightness.light),
      darkTheme: buildTheme(Brightness.dark),
      themeMode: mode,
      home: const SamplePage(),
    );
  }
}

class SamplePage extends StatelessWidget {
  const SamplePage({super.key});

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    return Scaffold(
      appBar: AppBar(title: const Text('Theming')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            Text('Balance', style: theme.textTheme.headlineMedium),
            // Read colours from the scheme instead of hard-coding them.
            Text('Overdue', style: TextStyle(color: theme.colorScheme.error)),
            const SizedBox(height: 16),
            FilledButton(onPressed: () {}, child: const Text('Pay now')),
            const SizedBox(height: 16),
            const TextField(decoration: InputDecoration(labelText: 'Amount')),
          ],
        ),
      ),
    );
  }
}
```

Three layers of theming in that file:

1. **`ColorScheme.fromSeed`** generates the colour roles — `primary`, `onPrimary`, `surface`,
   `onSurface`, `error`, containers and more — from one seed colour and a brightness.
2. **Component themes** (`filledButtonTheme`, `inputDecorationTheme`) restyle every widget of a
   kind: all filled buttons become 48 pixels tall with 12-pixel corners; all text fields get outlines.
3. **`textTheme`** adjusts the type scale. Styles you don't specify keep Material's defaults.

`MaterialApp` takes `theme`, `darkTheme` and `themeMode` (`system`, `light` or `dark`). With
`ThemeMode.system`, the app follows the device setting automatically.

Widgets read from the theme with `Theme.of(context)`: `SamplePage` uses
`theme.colorScheme.error` for "Overdue" rather than `Colors.red`, so it adapts to dark mode.

## What `fromSeed` actually produced

```dart title="theme_test.dart"
import 'dart:math' as math;
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hello/l1/theme.dart';

String hex(Color c) => '#${c.toARGB32().toRadixString(16).padLeft(8, '0').substring(2).toUpperCase()}';

double contrast(Color a, Color b) {
  final la = a.computeLuminance(), lb = b.computeLuminance();
  return (math.max(la, lb) + 0.05) / (math.min(la, lb) + 0.05);
}

void main() {
  test('one seed, two schemes', () {
    for (final b in Brightness.values) {
      final s = ColorScheme.fromSeed(seedColor: brandSeed, brightness: b);
      print('${b.name.padRight(5)} primary=${hex(s.primary)} onPrimary=${hex(s.onPrimary)} '
          'surface=${hex(s.surface)} onSurface=${hex(s.onSurface)} error=${hex(s.error)} '
          'contrast(onPrimary/primary)=${contrast(s.onPrimary, s.primary).toStringAsFixed(1)}:1 '
          'contrast(onSurface/surface)=${contrast(s.onSurface, s.surface).toStringAsFixed(1)}:1');
    }
    print('seed itself: ${hex(brandSeed)}');
  });

  for (final mode in [ThemeMode.light, ThemeMode.dark]) {
    testWidgets('widgets pick up the theme ($mode)', (tester) async {
      await tester.pumpWidget(ThemedApp(mode: mode));
      final ctx = tester.element(find.text('Pay now'));
      final scheme = Theme.of(ctx).colorScheme;
      final button = tester.renderObject<RenderBox>(find.byType(FilledButton));
      final material = tester.widget<Material>(
          find.descendant(of: find.byType(FilledButton), matching: find.byType(Material)));
      print('${mode.name.padRight(5)} button bg=${hex(material.color!)} (scheme.primary=${hex(scheme.primary)}) '
          'height=${button.size.height} radius=${(material.shape as RoundedRectangleBorder).borderRadius}');
    });
  }
}
```

```console
$ flutter test test/l1/theme_test.dart
dark  primary=#8CD5B3 onPrimary=#003826 surface=#0F1512 onSurface=#DEE4DE error=#FFB4AB contrast(onPrimary/primary)=7.7:1 contrast(onSurface/surface)=14.3:1
light primary=#1E6A4F onPrimary=#FFFFFF surface=#F5FBF5 onSurface=#171D1A error=#BA1A1A contrast(onPrimary/primary)=6.5:1 contrast(onSurface/surface)=16.3:1
seed itself: #0B6E4F
light button bg=#1E6A4F (scheme.primary=#1E6A4F) height=48.0 radius=BorderRadius.circular(12.0)
dark  button bg=#8CD5B3 (scheme.primary=#8CD5B3) height=48.0 radius=BorderRadius.circular(12.0)
00:00 +3: All tests passed!
```

(Flutter 3.44.8. The exact values come from Material's colour algorithm and may shift slightly between
Flutter releases.)

**Light and dark are different palettes, not inverted ones.** In dark mode `primary` became a light
mint (`#8CD5B3`) with a dark green `onPrimary` — colours chosen so text on a primary button stays
readable on a dark surface.

**The contrast is built in.** The `on…` colours are generated to contrast with their backgrounds:
6.5:1 and 7.7:1 for button text, 14–16:1 for body text, all above the WCAG AA threshold of 4.5:1 for
normal text. The `contrast` function in the test uses the WCAG formula (relative luminance via
`computeLuminance`), so you can assert accessibility in tests whenever you override colours by hand.

**`primary` is not your seed.** The seed was `#0B6E4F`; light `primary` came out `#1E6A4F`. `fromSeed`
extracts the seed's *hue and chroma* and builds tonal palettes from them, picking tones that meet the
contrast targets. If brand fidelity matters more, there's a variant for that:

```dart
ColorScheme.fromSeed(seedColor: brandSeed, dynamicSchemeVariant: DynamicSchemeVariant.fidelity)
```

```text
tonalSpot  primary=#1E6A4F
fidelity   primary=#00543B
```

Neither variant returns the seed exactly. If your brand guidelines require a specific hex value for a
specific role, set it explicitly with `.copyWith(primary: …)` — and then check its contrast with the
`on…` colour yourself, because you've stepped outside the algorithm's guarantees.

**Widgets used the theme.** The `FilledButton`'s background was `scheme.primary` in both modes, and the
component theme's 48-pixel height and 12-pixel radius applied without touching the button code.

## Rules for themeable code

- **Never hard-code colours in widgets.** Use `colorScheme` roles: `primary` for key actions,
  `secondary`/`tertiary` for accents, `surface`/`surfaceContainer…` for backgrounds, `error` for errors,
  and always the matching `on…` colour for content on top.
- **Use the text theme's roles** (`titleLarge`, `bodyMedium`, `labelSmall` …) and adjust with
  `.copyWith`, so a font change in one place changes the whole app.
- **Prefer component themes over per-widget `style:`.** A per-widget style is right for a genuine
  exception, not for the house style.
- **Test both modes.** A widget test per `ThemeMode`, as above, catches widgets that only look right in
  light mode.
- **Brand colours with meaning** (success green, warning amber) that Material doesn't define can be
  added as a `ThemeExtension`, so they also get light/dark variants.

## How It Actually Works

`MaterialApp` wraps your app in an `AnimatedTheme`, which places an inherited `Theme` widget at the top of
the tree. `Theme.of(context)` is an `InheritedWidget` lookup: it walks up to the nearest `Theme` and
registers the calling widget as a **dependent**, so when the theme changes (the user switches to dark
mode), exactly the widgets that read it rebuild — and `AnimatedTheme` interpolates the colours over a
short animation. Material widgets resolve their appearance in layers: an explicit `style:` argument
first, then the component theme (`FilledButtonThemeData`), then defaults computed from the
`ColorScheme` and `TextTheme`.

`ColorScheme.fromSeed` runs the Material Color Utilities algorithm: it converts the seed to the **HCT**
colour space (hue, chroma, tone — tone being perceptual lightness), builds tonal palettes for each role
from the seed's hue and chroma, and assigns specific tones to each role (for `tonalSpot` light, roughly
tone 40 for `primary` and 100 for `onPrimary`). Because tone tracks perceived lightness, fixed tone gaps
produce predictable contrast — that's where the ratios above come from.

## Common mistakes

- **`Colors.blue` and `Colors.white` sprinkled through widgets**, breaking dark mode.
- **Defining only `theme`** and forgetting `darkTheme`.
- **Overriding `primary` but not `onPrimary`**, producing unreadable buttons.
- **Copy-pasting `style:` on every button** instead of a component theme.
- **Expecting the seed colour to appear verbatim** as `primary`.

## Exercise

1. Add `cardTheme` so every `Card` has 0 elevation and a 1-pixel outline in `outlineVariant`.
2. Write a test that fails if `contrast(onPrimary, primary) < 4.5` for both modes, then set
   `primary` to a light yellow with `copyWith` and watch it fail.
3. Add a `ThemeExtension<StatusColors>` with `success` and `warning` colours that differ between light and
   dark, and use it from a widget via `Theme.of(context).extension<StatusColors>()`.
4. Add a switch that toggles `ThemeMode` at runtime. Where should that state live so the whole app sees it?
