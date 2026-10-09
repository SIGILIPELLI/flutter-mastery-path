---
description: "Flutter on the web and desktop: JS vs Wasm builds with real sizes, tests in Chrome, URL strategy, dart:io pitfalls and desktop conventions."
---

# 07 · Flutter on the Web & Desktop

The same Flutter codebase can target the web, Windows, macOS and Linux. "Can" is doing work in that sentence: each target
compiles and runs, but users expect different things from a website and a desktop app than from a phone app. This lesson builds
the Level 2 reading-list app for the web in both compilation modes, runs its tests in Chrome, hits the most common portability
bug on purpose, and covers what desktop needs.

## Web: two compilers, one app

```console
$ flutter build web --release -t lib/rl/main.dart
Compiling lib/rl/main.dart for the Web...
Wasm dry run succeeded. Consider building and testing your application with the `--wasm` flag. See docs for more info: https://docs.flutter.dev/platform-integration/web/wasm
Use --no-wasm-dry-run to disable these warnings.
Font asset "CupertinoIcons.ttf" was tree-shaken, reducing it from 257628 to 1472 bytes (99.4% reduction). ...
Font asset "MaterialIcons-Regular.otf" was tree-shaken, reducing it from 1645184 to 8076 bytes (99.5% reduction). ...
Compiling lib/rl/main.dart for the Web...                          37.7s
✓ Built build/web

$ flutter build web --release --wasm -t lib/rl/main.dart -o ../web_wasm
...
Compiling lib/rl/main.dart for the Web...                          35.7s
✓ Built ../web_wasm
```

Both builds above were run with Flutter 3.44.8 on the course machine. What they produced:

| | JavaScript build | `--wasm` build |
|---|---|---|
| Compiled app code | `main.dart.js`, 2,571,692 bytes (762,231 gzipped) | `main.dart.wasm`, 2,357,235 bytes (866,257 gzipped) + a 32,735-byte `main.dart.mjs` loader |
| Renderer | CanvasKit (Skia compiled to WebAssembly) | Skwasm where the browser supports WasmGC, falling back to the JS build otherwise |

Observations:

- **The default build already ran a "Wasm dry run"** — the tool checks whether your code (and dependencies) would compile to
  Wasm, and suggests trying it. Code that uses `dart:html` or old JS-interop APIs fails that check; the modern replacements
  are `package:web` and `dart:js_interop`.
- **Icon fonts were tree-shaken** by over 99 %: only the glyphs the app references are kept. If you construct `IconData`
  dynamically, the tool can't see which glyphs you need — use `--no-tree-shake-icons` or reference icons as constants.
- **The renderer is most of the download.** The whole `build/web` folder was 41 MB on disk, 37 MB of it the `canvaskit/` folder —
  several renderer variants (CanvasKit, Skwasm, a Chromium-specific CanvasKit) plus `.symbols` files for debugging. A browser
  downloads only the variant it uses, typically a few megabytes, and caches it. Still: a Flutter web app's first load is heavier
  than a typical website's.
- Gzipped, the Wasm module was *larger* than the JS here; Wasm's advantages are execution speed and consistency, not size.
  Measure your own app before deciding.

## Running tests in a browser

`flutter test --platform chrome` compiles tests to JavaScript and runs them in headless Chrome — the quickest way to catch
web-only breakage:

```console
$ flutter test --platform chrome test/rl/reading_list_test.dart
...
missing-book page shown
00:01 +6: All tests passed!
```

The reading-list app passed: `shared_preferences` has a web implementation (it uses the browser's `localStorage`), and nothing
in it touches the file system. The Level 3 habit tracker is a different story — its `LocalStore` writes files with `dart:io`:

```console
$ flutter test --platform chrome test/ht/habit_tracker_test.dart
  Unsupported operation: _Namespace
00:01 +0 -10: Some tests failed.
```

Every test failed, including the pure streak tests, because the shared `setUp` creates a temp directory. **`dart:io` file and
process APIs don't exist in browsers.** The fixes: keep I/O behind an interface (the habit tracker already has `LocalStore`) and
provide a web implementation (IndexedDB via a package, or `shared_preferences` for small data), selected with a conditional
import:

```dart
import 'local_store_io.dart' if (dart.library.js_interop) 'local_store_web.dart';
```

Also check `kIsWeb` (from `package:flutter/foundation.dart`) instead of `Platform.isX` — `dart:io`'s `Platform` throws on the web.

## Web-specific concerns

- **URLs**: Flutter web uses hash URLs (`/#/contacts/2`) by default. Call `usePathUrlStrategy()` (from
  `package:flutter_web_plugins/url_strategy.dart`) before `runApp` for clean paths — and configure your host to serve
  `index.html` for every path, or deep links 404 on refresh.
- **Hosting under a sub-path**: `flutter build web --base-href /myapp/`.
- **SEO**: Flutter web renders to a canvas; content isn't HTML text that crawlers index well. Use Flutter web for *apps*
  (dashboards, tools, internal software), and a regular website for marketing pages and content.
- **Text selection, right-click, browser find** behave differently from HTML; wrap selectable content in `SelectionArea`.
- **CORS**: the browser enforces it on HTTP calls; your API must send the right headers (lesson [Level 2 · 07](../level-2/07-http-json.md)).
- **Wasm and cross-origin isolation**: Skwasm can use multiple threads only when the page is served with
  `Cross-Origin-Opener-Policy` and `Cross-Origin-Embedder-Policy` headers; without them it runs single-threaded. Check the Flutter
  Wasm docs for the current requirements.

## Desktop

!!! warning "Not run here"
    Desktop builds use each OS's native toolchain — Xcode for macOS, Visual Studio for Windows, a C++ toolchain and GTK
    development libraries for Linux — and must be built on that OS. The course machine is a Mac without Xcode, so no desktop
    build was produced; `flutter devices` listed `macOS (desktop)`, but building for it failed for lack of `xcodebuild`
    (shown in [Level 3 · 09](../level-3/09-integration-tests.md)).

```bash
flutter create --platforms=windows,macos,linux .   # add desktop folders to an existing project
flutter run -d macos
flutter build macos|windows|linux --release
```

What desktop users expect, which mobile-first apps usually lack:

- **Resizable windows** — your responsive layouts from [Level 3 · 04](../level-3/04-responsive.md) at every width, including very
  wide. Packages like `window_manager` set minimum sizes and titles.
- **Keyboard first**: shortcuts (`Shortcuts`/`Actions`, `CallbackShortcuts`), visible focus, Tab order, Enter/Escape in dialogs.
- **Mouse**: hover states, right-click context menus (`ContextMenuRegion` patterns), scroll wheels and trackpads, cursors
  (`MouseRegion(cursor: SystemMouseCursors.click)`).
- **Native menus**: `PlatformMenuBar` provides the macOS menu bar.
- **Dense layouts**: `VisualDensity.compact` (Material adjusts automatically on desktop platforms).
- **Distribution**: macOS apps need signing and notarization (and sandbox entitlements — network access is *off* by default in
  the macOS sandbox, so HTTP calls fail until you add `com.apple.security.network.client`); Windows apps are typically
  packaged as MSIX; Linux as Snap, Flatpak, or distro packages.

## How It Actually Works

On the web, Dart is compiled either to JavaScript (`dart2js`, with tree shaking and minification) or to WebAssembly with the GC
proposal (`dart2wasm`). The Flutter framework is the same Dart code as on mobile; what changes is the engine. Instead of the C++
engine, the web build ships a renderer compiled to WebAssembly — CanvasKit (Skia) or Skwasm — that draws the layer tree into a
`<canvas>` via WebGL, plus a DOM-based "semantics" layer so screen readers and some browser features can see the content. The
bootstrap script (`flutter_bootstrap.js`) detects browser capabilities and loads the right combination. Platform channels map to
JavaScript implementations of plugins, which is why a plugin needs a web implementation to work there. Desktop builds, by contrast,
use the regular C++ engine and AOT-compiled Dart, hosted in a native window created by a small per-OS "runner" project — the
`macos/`, `windows/` and `linux/` folders — which you can edit like any native app.

## Common mistakes

- **`dart:io` in shared code** — breaks the web build or the web runtime. Hide it behind interfaces and conditional imports.
- **`Platform.isAndroid` checks** that throw on the web. Use `defaultTargetPlatform` / `kIsWeb`.
- **Forgetting server rewrites** for path URLs, so refresh gives a 404.
- **Shipping a phone layout to desktop**: a 400-pixel column in a 1,600-pixel window.
- **macOS sandbox network entitlement** missing, so release builds can't reach the API.
- **Judging web performance in debug mode** (`flutter run -d chrome` uses a development compiler); test release builds.

## Exercise

1. Give the habit tracker a web `LocalStore` backed by `shared_preferences` and a conditional import. Make
   `flutter test --platform chrome test/ht/` pass.
2. Switch the reading-list app to `usePathUrlStrategy()` and write the rewrite rule for your hosting provider of choice.
3. Build both JS and Wasm versions of your own app and compare sizes and startup in Chrome DevTools' network and performance tabs.
4. On a machine with a desktop toolchain, run the Level 3 responsive shell as a desktop app, add `Ctrl/⌘+F` to focus a search
   field, and set a minimum window size.
