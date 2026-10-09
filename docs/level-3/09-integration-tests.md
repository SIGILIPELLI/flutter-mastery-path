---
description: "Flutter integration tests: what they prove that widget tests can't, a restart-persistence flow, running on devices and CI, and test strategy."
---

# 09 · Integration Tests & Test Strategy

Widget tests run your real widgets on the Dart VM with fake plugins, fake time and no GPU. They're fast, deterministic, and
should be most of your UI tests. But some bugs only exist on a real device: a plugin's native side, platform permissions,
real storage surviving a restart, keyboard and system-UI interactions, release-mode compilation, rendering on an actual GPU.
**Integration tests** run your app on a device, emulator, simulator, or desktop and drive it with the same `WidgetTester` API.

## Setup

```bash
flutter pub add 'dev:integration_test:{"sdk":"flutter"}'
```

Tests go in a top-level `integration_test/` folder (not `test/`).

## A flow that needs the real platform

The [Level 2 project](../level-2/10-project-reading-list.md) stores books with `shared_preferences`. Its widget tests used an
in-memory fake, so they never proved that a book survives in the *real* platform store. This test does:

```dart title="integration_test/reading_list_flow_test.dart"
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:l2/rl/app.dart';
import 'package:l2/rl/repository.dart';
import 'package:l2/rl/state.dart';
import 'package:shared_preferences/shared_preferences.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('add a book, restart the app, it is still there', (tester) async {
    // Real plugin, real platform storage: start from a clean slate.
    final prefs = await SharedPreferences.getInstance();
    await prefs.clear();

    Future<void> launch() async {
      await tester.pumpWidget(const SizedBox()); // tear down any previous app
      await tester.pumpWidget(ProviderScope(
        overrides: [repositoryProvider.overrideWithValue(PrefsBookRepository(prefs))],
        child: const ReadingListApp(),
      ));
      await tester.pumpAndSettle();
    }

    await launch();
    await tester.tap(find.byTooltip('Add book'));
    await tester.pumpAndSettle();
    await tester.enterText(find.byKey(const Key('title')), 'Piranesi');
    await tester.enterText(find.byKey(const Key('author')), 'Susanna Clarke');
    await tester.tap(find.text('Save'));
    await tester.pumpAndSettle();
    expect(find.text('Piranesi'), findsOneWidget);

    await launch(); // a fresh ProviderScope re-reads storage
    expect(find.text('Piranesi'), findsOneWidget);
    expect(find.text('To read (1)'), findsOneWidget);
  });
}
```

The differences from a widget test are small but important:

- `IntegrationTestWidgetsFlutterBinding.ensureInitialized()` replaces the test binding: frames are driven by the real engine,
  plugins talk to real native code, and results are reported back to the host machine.
- No `setMockInitialValues`: `SharedPreferences` writes to Android's or iOS's actual store, so the test clears it first to be
  repeatable.
- "Restart" here means tearing down the widget tree and building a new `ProviderScope`, which forces a fresh read from storage.
  (A true process restart can't happen inside one test run; for cold-start behaviour, run the app twice from a script or use a
  device-farm feature.)

## Running it

```bash
flutter test integration_test/reading_list_flow_test.dart            # picks a connected device
flutter test integration_test -d emulator-5554                        # a specific device
flutter test integration_test -d macos                                # desktop target
```

!!! warning "What happened when this course ran it"
    The machine used to write this course has no usable Android build setup (SDK licenses not accepted, no emulator), no Xcode and no
    simulators, so this integration test
    could **not** be run on a device. The actual attempts, verbatim:

    ```console
    $ flutter test integration_test/reading_list_flow_test.dart
    Web devices are not supported for integration tests yet.

    $ flutter test integration_test/reading_list_flow_test.dart -d macos
    Failed to load ".../integration_test/reading_list_flow_test.dart": No macOS desktop project configured. See https://flutter.dev/to/add-desktop-support to learn about adding macOS support to a project.

    $ flutter create --platforms=macos .
    $ flutter test integration_test/reading_list_flow_test.dart -d macos
    Failed to load ".../integration_test/reading_list_flow_test.dart":
    Xcode failed to resolve Swift Package Manager dependencies:
    xcrun: error: unable to find utility "xcodebuild", not a developer tool or in PATH
    ```

    What *was* verified: `flutter analyze integration_test` reported "No issues found!", and the same test body run as an ordinary
    widget test (with `SharedPreferences.setMockInitialValues({})` instead of the integration binding) passed. Those errors are
    also worth recognizing: integration tests need a target platform folder in the project *and* that platform's toolchain.

On the **web**, `flutter test` doesn't run integration tests; the `flutter drive` command with a ChromeDriver can, using a small
driver file (`test_driver/integration_test.dart`) — see the `integration_test` package docs for the current setup.

## Running on CI and device farms

- **Android emulators on CI**: GitHub Actions Linux runners can boot an Android emulator (several community actions exist);
  expect each run to take minutes.
- **iOS**: needs a macOS runner with Xcode and a simulator.
- **Device farms** (Firebase Test Lab, AWS Device Farm, others) run the integration test as an instrumented app on many real
  devices; the `integration_test` package documents how to build the APK/IPA bundles they expect.

Because they're slow and occasionally flaky, run integration tests on merges to main or nightly, not on every keystroke. CI
setup is in [Level 4 · 05](../level-4/05-ci.md).

## A test strategy that holds up

Think of a pyramid, wide at the bottom:

| Layer | What | How many | Example from this course |
|---|---|---|---|
| Unit | pure logic, models, parsing, notifiers | many | `passwordScore`, `calculateSplit`, `ReadingList` rules |
| Widget | screens and flows with fakes | many | signup form, reading-list add/move/rate/remove |
| Golden | appearance of key components | a few | strength meter, sparkline |
| Integration | real platform, end-to-end | a handful of critical journeys | add book → restart → still there |

Rules of thumb:

- **Each bug fix gets a test at the lowest layer that reproduces it.**
- **Integration tests cover journeys, not features**: sign-up → first action → restart, purchase flow, offline → online.
- **Make the app testable by design**: dependency seams (repositories, `Provider` overrides) are what let widget tests replace
  the network and storage. Everything earlier in this course was set up so that this table is possible.
- **Measure**: `flutter test --coverage` writes `coverage/lcov.info`; use it to find untested logic, not as a target to game.

## Performance tests

`integration_test` can also record performance: `binding.traceAction(() async { ... }, reportKey: 'scrolling')` captures a
timeline of frame build and raster times while the action runs, which a driver script can summarise (frame time percentiles,
missed frames). Run these in **profile mode** on a real device — debug mode numbers are meaningless for performance. See
[Level 4 · 02](../level-4/02-performance.md).

## How It Actually Works

`flutter test integration_test/...` builds your test file as the app's entry point (instead of `lib/main.dart`), installs it on
the device, and launches it. `IntegrationTestWidgetsFlutterBinding` extends the live binding: `pump` waits for real frames
(the default policy schedules frames as the app would), and `pumpAndSettle` waits on real time. Each test's result is collected
by the binding and sent back to the `flutter` tool on your machine over a platform channel and the VM service connection, which
is why you see ordinary `+1: All tests passed!` output on the host even though the code ran on a phone. On Android, the same
test can also run inside a native instrumentation test (`flutter build apk --debug` plus an `androidTest` runner), which is
what device farms use.

## Common mistakes

- **Writing everything as integration tests**: slow suites that nobody runs.
- **Depending on device state** (leftover storage, logged-in accounts, locale) — reset what you use at the start of each test.
- **Fixed `Future.delayed` waits** for network or animations. Prefer `pumpAndSettle`, or poll for a finder with a timeout.
- **Hitting production servers** from tests. Point the app at a test backend with `--dart-define` ([Level 4 · 04](../level-4/04-flavors-config.md)).
- **Measuring performance in debug mode.**

## Exercise

1. On a device or emulator you have, run the integration test. Then remove the `prefs.clear()` line and run it twice. What
   happens on the second run, and why?
2. Add a journey: add three books, move one to *Reading*, rate one, restart, and verify all three shelves.
3. Wrap a long scroll of 500 books in `binding.traceAction` and print the summary in profile mode.
4. Set up a GitHub Actions job that runs only the widget and unit tests on every push, and the integration test nightly on an
   Android emulator.
