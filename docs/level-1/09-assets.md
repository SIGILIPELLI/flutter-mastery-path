---
description: "Bundle assets in Flutter: images, JSON and fonts in pubspec, resolution-aware 2.0x/3.0x images verified in tests, and rootBundle loading."
---

# 09 · Assets, Images & Fonts

Apps ship with files: icons and illustrations, bundled JSON, custom fonts. Flutter packs them into
an **asset bundle** inside the app and gives you APIs to load them. The rules are simple but easy
to get subtly wrong — a missing `pubspec.yaml` line, an image that looks blurry on high-density
screens, a font family name that silently falls back. This lesson bundles all three kinds, then
verifies in tests which file is actually loaded at each screen density and what failure looks like.

## Declaring assets

Nothing in your project folder is bundled unless `pubspec.yaml` says so:

```yaml title="pubspec.yaml (flutter section)"
flutter:
  uses-material-design: true
  assets:
    - assets/images/
    - assets/data/quotes.json
  fonts:
    - family: Brand
      fonts:
        - asset: assets/fonts/Roboto-Regular.ttf
        - asset: assets/fonts/Roboto-Medium.ttf
          weight: 500
```

- A path ending in `/` includes every file **directly** in that folder. Subfolders need their own
  entry — except resolution variant folders like `2.0x/`, which Flutter picks up automatically.
- A font **family** is a name you choose; each file in it is one weight/style. Here `Brand` has a
  regular file and a medium (500) file. (The Roboto files used here come from the Flutter SDK's
  bundled Material fonts, under an open licence; for your own fonts, check the licence permits
  embedding in apps.)

The folder layout:

```text
assets/
├── data/quotes.json
├── fonts/Roboto-Regular.ttf, Roboto-Medium.ttf
└── images/
    ├── badge.png          48×48   (1.0x)
    ├── 2.0x/badge.png     96×96
    └── 3.0x/badge.png     144×144
```

(The three test PNGs were generated as flat red, green and blue squares, so it's obvious which one
loaded.)

## Using them

```dart title="assets_demo.dart"
import 'dart:convert';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart' show rootBundle;

class Quote {
  const Quote(this.text, this.author);
  final String text;
  final String author;
}

Future<List<Quote>> loadQuotes([AssetBundle? bundle]) async {
  final raw = await (bundle ?? rootBundle).loadString('assets/data/quotes.json');
  final list = jsonDecode(raw) as List<dynamic>;
  return [for (final q in list) Quote(q['text'] as String, q['author'] as String)];
}

class QuoteCard extends StatelessWidget {
  const QuoteCard({super.key, required this.quote});
  final Quote quote;

  @override
  Widget build(BuildContext context) {
    return Card(
      child: ListTile(
        leading: Image.asset('assets/images/badge.png', width: 48, height: 48),
        title: Text(quote.text, style: const TextStyle(fontFamily: 'Brand')),
        subtitle: Text(quote.author,
            style: const TextStyle(fontFamily: 'Brand', fontWeight: FontWeight.w500)),
      ),
    );
  }
}
```

- `rootBundle.loadString` reads a text asset. It's `async` — assets may be compressed or read from
  disk — so loading belongs in a `Future` (shown with `FutureBuilder` in
  [Level 2 · 06](../level-2/06-async-ui.md)), not in `build`. Passing an `AssetBundle` parameter lets
  tests supply a fake bundle if needed.
- `Image.asset('assets/images/badge.png', width: 48, height: 48)` names the **1.0x** file; Flutter
  chooses a variant for the screen.
- `fontFamily: 'Brand'` with `fontWeight: FontWeight.w500` selects the medium file.

## Verifying what loads

```dart title="assets_test.dart"
import 'dart:ui' as ui;
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hello/l1/assets_demo.dart';

void main() {
  test('JSON asset loads and parses', () async {
    TestWidgetsFlutterBinding.ensureInitialized();
    final quotes = await loadQuotes();
    print('quotes: ${quotes.map((q) => q.author).toList()}');
  });

  test('the asset manifest lists resolution variants', () async {
    TestWidgetsFlutterBinding.ensureInitialized();
    final manifest = await AssetManifest.loadFromAssetBundle(rootBundle);
    final variants = manifest.getAssetVariants('assets/images/badge.png')!;
    print('variants: ${variants.map((v) => '${v.key} @${v.targetDevicePixelRatio ?? 1.0}x').toList()}');
    print('all assets: ${manifest.listAssets()..sort()}');
  });

  for (final dpr in [1.0, 2.0, 3.0]) {
    testWidgets('picks the right image at ${dpr}x', (tester) async {
      tester.view.devicePixelRatio = dpr;
      addTearDown(tester.view.resetDevicePixelRatio);
      await tester.pumpWidget(const MaterialApp(home: Scaffold(body: QuoteCard(
          quote: Quote('Simplicity is prerequisite for reliability.', 'Edsger W. Dijkstra')))));
      await tester.runAsync(() async {
        final image = tester.widget<Image>(find.byType(Image));
        final key = await (image.image as AssetImage).obtainKey(createLocalImageConfiguration(
            tester.element(find.byType(Image))));
        final data = await rootBundle.load(key.name);
        final codec = await ui.instantiateImageCodec(data.buffer.asUint8List());
        final frame = await codec.getNextFrame();
        print('dpr=$dpr -> ${key.name} (scale ${key.scale}), decoded ${frame.image.width}x${frame.image.height}px, '
            'shown at ${tester.getSize(find.byType(Image))} logical px');
      });
    });
  }

  testWidgets('missing asset fails at load time, not compile time', (tester) async {
    await tester.pumpWidget(MaterialApp(home: Image.asset('assets/images/nope.png')));
    await tester.runAsync(() => Future<void>.delayed(const Duration(milliseconds: 50)));
    await tester.pump();
    final e = tester.takeException();
    print('missing asset: ${e.runtimeType}: ${e.toString().split('\n').first}');
  });

  testWidgets('custom font: registered vs fallback', (tester) async {
    final loader = FontLoader('Brand')..addFont(rootBundle.load('assets/fonts/Roboto-Regular.ttf'));
    await loader.load();
    await tester.pumpWidget(const Directionality(textDirection: TextDirection.ltr, child: Center(
      child: Column(mainAxisSize: MainAxisSize.min, children: [
        Text('Hello', style: TextStyle(fontFamily: 'Brand', fontSize: 20)),
        Text('Hello', style: TextStyle(fontSize: 20)),
      ]),
    )));
    final sizes = tester.renderObjectList<RenderBox>(find.text('Hello')).map((r) => r.size).toList();
    print('Brand font: ${sizes[0]}   test default font: ${sizes[1]}');
  });
}
```

```console
$ flutter test test/l1/assets_test.dart
quotes: [Edsger W. Dijkstra, Donald Knuth]
variants: [assets/images/badge.png @1.0x, assets/images/2.0x/badge.png @2.0x, assets/images/3.0x/badge.png @3.0x]
all assets: [assets/data/quotes.json, assets/fonts/Roboto-Medium.ttf, assets/fonts/Roboto-Regular.ttf, assets/images/badge.png, packages/cupertino_icons/assets/CupertinoIcons.ttf]
dpr=1.0 -> assets/images/badge.png (scale 1.0), decoded 48x48px, shown at Size(48.0, 48.0) logical px
dpr=2.0 -> assets/images/2.0x/badge.png (scale 2.0), decoded 96x96px, shown at Size(48.0, 48.0) logical px
dpr=3.0 -> assets/images/3.0x/badge.png (scale 3.0), decoded 144x144px, shown at Size(48.0, 48.0) logical px
missing asset: FlutterError: Unable to load asset: "assets/images/nope.png".
Brand font: Size(46.0, 23.0)   test default font: Size(100.0, 20.0)
00:00 +7: All tests passed!
```

### Resolution-aware images

Flutter sizes things in **logical pixels**. A phone with a device pixel ratio of 3 has three physical
pixels per logical pixel in each direction. The image is always shown at 48 × 48 *logical* pixels, but
at 3x Flutter loaded the 144 × 144 file, so every physical pixel has image data and it looks sharp. With
only a 1x file, a 3x screen would upscale 48 pixels to 144 and look blurry. Provide 2.0x and 3.0x
variants for raster images (or use vector formats where possible).

The asset manifest shows the mechanism: the build tool records each image's variants with their
ratios. Note `listAssets()` lists only the main `badge.png` — variants are attached to it, not separate
assets. `packages/cupertino_icons/...` is there because the template depends on that package; package
assets are namespaced by package name.

### Missing assets fail at runtime

A typo in an asset path compiles fine and fails when loading, with `Unable to load asset`. Two habits
catch this early: keep asset paths in one place (a class of `static const` strings, or a code generator
like `flutter_gen`), and have a test that loads every declared asset.

### Fonts in tests

Two `Text('Hello')` widgets at size 20 measured differently: the `Brand` (Roboto) text was 46 × 23, the
default text 100 × 20. As [lesson 01](01-what-flutter-is.md) noted, the test environment's default font
draws every character as a box the size of the font (5 characters × 20 = 100). Bundled fonts are *not*
loaded into tests automatically; the test used `FontLoader` to register `Brand`. That's also how
**golden tests** ([Level 2 · 09](../level-2/09-testing.md)) render real text instead of boxes.

## How It Actually Works

At build time, the Flutter tool reads the `flutter:` section of `pubspec.yaml`, copies each listed file
into the app bundle (`flutter_assets/` inside the APK, IPA or web build), discovers variant folders like
`2.0x/`, and writes an **asset manifest** (and a font manifest) describing them. At run time `rootBundle`
reads from that bundle through the platform embedder.

`Image.asset` creates an `AssetImage` provider. When the image resolves, it receives an
`ImageConfiguration` that includes the device pixel ratio (from `MediaQuery`), looks the asset up in the
manifest, picks the variant whose ratio is closest to the device's (preferring higher ratios when between
two), and returns an `AssetBundleImageKey` with that file name and its `scale`. The decoded image is then
drawn at `pixels / scale` logical pixels — 144 / 3 = 48 — and cached in the `ImageCache` keyed by that
`AssetBundleImageKey`, so the same file isn't decoded twice. Fonts declared in the pubspec are registered
with the engine at startup from the font manifest; `fontFamily` then selects them by name, and the engine
picks the file whose weight and style best match the `TextStyle`.

## Common mistakes

- **Forgetting the pubspec entry** (or the trailing `/` on a folder, or the subfolder).
- **Only 1x raster images**, which look soft on modern phones.
- **Loading assets in `build`** instead of once in state or a `FutureBuilder`'s future created outside
  `build`.
- **Misspelled font family names** — Flutter silently falls back to the default font.
- **Huge images decoded at full size** for small thumbnails — use `cacheWidth`/`cacheHeight` on
  `Image.asset` to decode smaller.

## Exercise

1. Remove the `2.0x` variant and re-run the 2.0x test. Which file is chosen now, and at what scale?
2. Write a test that reads the asset manifest and loads every listed asset, failing on any error.
3. Add `cacheWidth: 48` to the `Image.asset` and measure the decoded size at 3x. When is that a good idea?
4. Add an italic font file to the `Brand` family (`style: italic`) and confirm with a test which file a
   `TextStyle(fontStyle: FontStyle.italic, fontFamily: 'Brand')` uses.
