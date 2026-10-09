---
description: "Install and check Flutter, create a project, learn every file that matters, read the template app, and run tests, the analyzer and a web build."
---

# 02 · Setup, flutter create & Project Anatomy

This lesson gets a project on disk and walks through it file by file, then runs the four
commands you'll use every day: `flutter doctor`, `flutter test`, `flutter analyze` and
`flutter build`. Everything was run with **Flutter 3.44.8** (Dart 3.12.2) on an Apple-silicon
Mac. Install instructions change with each OS and release, so follow the official guide at
[docs.flutter.dev/get-started](https://docs.flutter.dev/get-started/install) for that part;
this lesson starts once `flutter` is on your `PATH`.

## `flutter doctor`: what can this machine build?

```console
$ flutter doctor
Doctor summary (to see all details, run flutter doctor -v):
[✓] Flutter (Channel stable, 3.44.8, on macOS 27.0.1 26A434 darwin-arm64, locale en-IN)
[!] Android toolchain - develop for Android devices (Android SDK version 36.0.0)
    ! Some Android licenses not accepted. To resolve this, run: flutter doctor --android-licenses
[!] Xcode - develop for iOS and macOS
    ✗ Xcode installation is incomplete; a full installation is necessary for iOS and macOS development.
    ...
    ! CocoaPods not installed.
[✓] Chrome - develop for the web
[✓] Connected device (2 available)
[✓] Network resources

! Doctor found issues in 2 categories.
```

That's the real state of the machine this course was written on, and it's worth reading
carefully because it shapes what you can do:

- **Android**: the SDK is installed but its licenses haven't been accepted, so Android builds
  won't run until someone runs `flutter doctor --android-licenses` and accepts them. (That's a
  legal agreement for *you* to read and accept, not something to script away.)
- **iOS and macOS**: these require a full Xcode install, which only exists on macOS.
- **Web**: ready.

You don't need every platform to learn Flutter. **Widget tests run on any machine without an
emulator**, and most of this course is verified that way. Where a lesson needs a device that
wasn't available, it says so.

## Creating a project

```console
$ flutter create --org dev.masterypath --platforms=android,ios,web demo
Creating project demo...
Wrote 82 files.
All done!
```

`--org` sets the reverse-domain prefix used for the Android application id and iOS bundle id
(`dev.masterypath.demo`). Choose it once; changing it after publishing is painful.
`--platforms` limits which platform folders are generated — you can add more later by running
`flutter create --platforms=macos .` inside the project.

## What's in the box

```text
demo/
├── lib/main.dart            your Dart code starts here
├── test/widget_test.dart    a widget test for the template app
├── pubspec.yaml             name, version, dependencies, assets, fonts
├── pubspec.lock             exact resolved versions (commit it for apps)
├── analysis_options.yaml    lint rules for `flutter analyze`
├── android/                 Gradle project that hosts the Flutter engine on Android
├── ios/                     Xcode project (Runner) that hosts it on iOS
├── web/                     index.html, manifest, icons for the web build
├── .metadata                which Flutter revision created the project
└── .gitignore               already excludes build/ and .dart_tool/
```

Ninety percent of your time is in `lib/` and `test/`. The platform folders are real native
projects: you open `ios/Runner.xcworkspace` in Xcode for signing, or edit
`android/app/build.gradle.kts` for the application id and SDK levels. You'll touch them in
[Level 4 · 06](../level-4/06-publishing.md).

The `pubspec.yaml` without its comments:

```yaml title="pubspec.yaml"
name: demo
description: "A new Flutter project."
publish_to: 'none' # Remove this line if you wish to publish to pub.dev
version: 1.0.0+1
environment:
  sdk: ^3.12.2
dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.8
dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^6.0.0
flutter:
  uses-material-design: true
```

- `version: 1.0.0+1` is *version name* + *build number*. Stores require the build number to
  increase with every upload.
- `environment.sdk` is the Dart SDK range the project accepts.
- `flutter` and `flutter_test` come from the SDK itself, not from pub.dev — hence `sdk: flutter`.
- `uses-material-design: true` bundles the Material icon font.

## Reading the template app

The generated `lib/main.dart`, with comments removed:

```dart title="lib/main.dart"
import 'package:flutter/material.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        colorScheme: .fromSeed(seedColor: Colors.deepPurple),
      ),
      home: const MyHomePage(title: 'Flutter Demo Home Page'),
    );
  }
}

class MyHomePage extends StatefulWidget {
  const MyHomePage({super.key, required this.title});
  final String title;

  @override
  State<MyHomePage> createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  int _counter = 0;

  void _incrementCounter() {
    setState(() {
      _counter++;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        backgroundColor: Theme.of(context).colorScheme.inversePrimary,
        title: Text(widget.title),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: .center,
          children: [
            const Text('You have pushed the button this many times:'),
            Text('$_counter', style: Theme.of(context).textTheme.headlineMedium),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: _incrementCounter,
        tooltip: 'Increment',
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

The whole course is in miniature here:

- `MaterialApp` sets up theming, navigation and localisation defaults; `Scaffold` provides the app
  bar, body and floating button slots.
- `MyApp` is **stateless** — it only describes. `MyHomePage` is **stateful**: its `State` object
  holds `_counter`, and `setState` tells Flutter to rebuild ([lesson 05](05-stateful-widgets.md)).
- `widget.title` is how a `State` reads its widget's configuration.
- `Theme.of(context)` looks up the nearest theme above this widget — the `BuildContext` lookup
  pattern you'll use constantly ([Level 3 · 01](../level-3/01-element-tree.md)).

Two syntax notes. `.fromSeed(...)` and `.center` are **dot shorthands**, a recent Dart feature:
where the expected type is known (`ColorScheme`, `MainAxisAlignment`), you can drop the type name.
Older code writes `ColorScheme.fromSeed(...)` and `MainAxisAlignment.center`; both forms work on
this Dart version, and you'll see the long form in most existing code. And `const` in front of
constructors lets Dart create the object once at compile time and reuse it — Flutter can then skip
rebuilding that subtree entirely.

## The daily commands

### Tests

```console
$ flutter test
00:00 +1: All tests passed!
```

The template test pumps `MyApp`, taps the `+` icon, calls `tester.pump()` to run a frame, and
checks the text changed from `0` to `1`. No device or emulator involved — this is the headless test
environment from [lesson 01](01-what-flutter-is.md).

### The analyzer

```console
$ flutter analyze
Analyzing demo...
No issues found! (ran in 5.0s)
```

With a `print` call added to `main`, `const` removed from `MyApp()`, and an unused local variable:

```console
$ flutter analyze
   info • Don't invoke 'print' in production code. Try using a logging framework • lib/main.dart:4:3 • avoid_print
warning • The value of the local variable 'unused' isn't used. Try removing the variable or using it • lib/main.dart:69:12 • unused_local_variable

2 issues found. (ran in 4.4s)
```

Note what *wasn't* reported: the missing `const`. The `flutter_lints` 6.0.0 rule set didn't flag
it in this run. If you want that check, enable `prefer_const_constructors` under `linter: rules:` in
`analysis_options.yaml`. Run the analyzer in CI and treat warnings as failures.

### Running and hot reload

`flutter run` builds a debug version for a connected device and attaches. This machine listed two:

```console
$ flutter devices
Found 2 connected devices:
  macOS (desktop) • macos  • darwin-arm64   • macOS 27.0.1 26A434 darwin-arm64
  Chrome (web)    • chrome • web-javascript • Google Chrome 154.0.8037.98
```

In a running session, press `r` for **hot reload** (inject changed code, keep state — change the
seed colour and the counter stays where it was), `R` for **hot restart** (rerun `main`, state lost),
and `q` to quit. Hot reload is interactive, so it isn't reproduced as output here; try it yourself.
It can't apply some changes — new `main` logic, changed field initialisers of already-created
`State` objects, enum changes — and the tool tells you when a restart is needed.

### A release build

```console
$ flutter build web --release
Font asset "MaterialIcons-Regular.otf" was tree-shaken, reducing it from 1645184 to 7800 bytes (99.5% reduction). Tree-shaking can be disabled by providing the --no-tree-shake-icons flag when building your app.
Compiling lib/main.dart for the Web...                             18.7s
✓ Built build/web
```

The output folder was 40 MB, but most of that is alternative rendering engines (`canvaskit/` held
37 MB across variants such as `canvaskit.wasm`, `skwasm.wasm` and a `chromium/` build); a browser
downloads only the one it needs. The app code itself, `main.dart.js`, was 1.9 MB. Icon
**tree-shaking** removed every Material icon the app doesn't use — 1.6 MB down to 7.8 KB. Mobile
release builds (`flutter build apk`, `flutter build ipa`) weren't run here for the toolchain reasons
above.

## How It Actually Works

`flutter create` copies a template and substitutes names. `flutter pub get` (run automatically by
most commands) resolves `pubspec.yaml` constraints against pub.dev, writes the exact versions to
`pubspec.lock`, and records package locations in `.dart_tool/package_config.json`, which is how
`import 'package:demo/main.dart'` finds files.

`flutter test` compiles your test with the Dart VM and runs it against `flutter_tester`, a headless
build of the engine with a fake window. `flutter run` in debug mode compiles Dart to **kernel**
(an intermediate form) and runs it on the Dart VM inside your app with a JIT; hot reload works by
sending only changed kernel to the running VM, which swaps function bodies and asks the framework to
rebuild every widget — `build` methods run again, but `State` objects survive. Release builds
compile ahead-of-time to native ARM/x64 code (mobile and desktop) or to JavaScript/WebAssembly (web),
with no JIT and no hot reload, and with unused code — including unused icons — removed.

## Common mistakes

- **Ignoring `flutter doctor`**, then fighting cryptic build errors that it already explained.
- **Not committing `pubspec.lock`** for an app, so CI and teammates get different package versions.
- **Changing `--org` / the bundle id after publishing.** Stores treat it as a different app.
- **Expecting hot reload to apply everything.** When behaviour looks stale, hot restart.
- **Reading the web build's folder size as download size.**

## Exercise

1. Create a project with `--platforms=web` only, then add Android support to it later with
   `flutter create --platforms=android .`. Which files appear?
2. Change the template so the button decrements, update the test, and run it.
3. Enable `prefer_const_constructors` in `analysis_options.yaml` and run `flutter analyze` on the
   template. How many suggestions do you get?
4. Run `flutter build web --release` and find which file in `build/web` your browser actually
   downloads first (open the network tab when serving it with any static server).
