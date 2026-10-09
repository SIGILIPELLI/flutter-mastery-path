---
description: "Configure Flutter builds per environment: --dart-define and --dart-define-from-file, a validated AppConfig, native flavors, and secrets."
---

# 04 · Configuration, Flavors & --dart-define

A real app talks to different backends in development, staging and production, logs differently, and often installs
side-by-side as separate apps ("MyApp Dev" next to "MyApp"). Flutter offers two mechanisms, which solve different halves of the
problem:

| Mechanism | Changes | Set with |
|---|---|---|
| **Dart defines** | values your Dart code reads at compile time (API URL, feature switches) | `--dart-define=KEY=value`, `--dart-define-from-file=config/dev.json` |
| **Flavors** | the *native* app: bundle/application id, app name, icon, signing, native SDK keys | Android `productFlavors`, iOS/macOS Xcode schemes; `--flavor dev` |

Most apps use both: a flavor per environment for the native side, and a matching defines file for the Dart side.

## Dart defines and a validated config object

```dart title="lib/config.dart"
/// Compile-time configuration. Values come from --dart-define / --dart-define-from-file.
/// `const` is required: these are resolved by the compiler, not read at runtime.
enum Environment { dev, staging, prod }

class AppConfig {
  const AppConfig._({required this.env, required this.apiUrl, required this.enableLogging, required this.sampleRate});

  factory AppConfig.fromEnvironment() {
    const envName = String.fromEnvironment('ENV', defaultValue: 'dev');
    const apiUrl = String.fromEnvironment('API_URL');
    const logging = bool.fromEnvironment('LOGGING', defaultValue: true);
    const sampleRate = int.fromEnvironment('SAMPLE_RATE_PERCENT', defaultValue: 100);

    final env = Environment.values.asNameMap()[envName] ??
        (throw ArgumentError.value(envName, 'ENV', 'must be one of ${Environment.values.map((e) => e.name)}'));
    if (apiUrl.isEmpty) throw StateError('API_URL is not set. Pass --dart-define-from-file=config/${env.name}.json');
    final uri = Uri.parse(apiUrl);
    if (env == Environment.prod && uri.scheme != 'https') throw StateError('prod must use https, got $apiUrl');
    return AppConfig._(env: env, apiUrl: uri, enableLogging: logging, sampleRate: sampleRate);
  }

  final Environment env;
  final Uri apiUrl;
  final bool enableLogging;
  final int sampleRate;

  @override
  String toString() => 'AppConfig(${env.name}, $apiUrl, logging=$enableLogging, sample=$sampleRate%)';
}
```

```json title="config/dev.json"
{
  "ENV": "dev",
  "API_URL": "http://localhost:8080/",
  "LOGGING": true
}
```

```json title="config/prod.json"
{
  "ENV": "prod",
  "API_URL": "https://api.example.com/",
  "LOGGING": false,
  "SAMPLE_RATE_PERCENT": 10
}
```

Key points:

- `String.fromEnvironment`, `bool.fromEnvironment` and `int.fromEnvironment` must be used as **`const`**. The compiler
  substitutes the values into the code; in release builds, code behind a `const false` condition can be removed entirely.
- **Validate once, at startup**, and fail loudly. A misconfigured build that crashes on launch is found in minutes; one that
  silently talks to the wrong server is found by users.
- One file per environment, checked into the repo (they contain no secrets — see below).

A small test that just prints the resolved config, run with different flags:

```dart title="test/config_test.dart"
import 'package:flutter/services.dart' show appFlavor;
import 'package:flutter_test/flutter_test.dart';
import 'package:l2/m/config.dart';

void main() {
  test('resolve config', () {
    try {
      print(AppConfig.fromEnvironment());
    } catch (e) {
      print('config error: $e');
    }
    print('appFlavor: $appFlavor');
  });
}
```

```console
$ flutter test test/cfg
config error: Bad state: API_URL is not set. Pass --dart-define-from-file=config/dev.json
appFlavor: null

$ flutter test test/cfg --dart-define-from-file=config/dev.json
AppConfig(dev, http://localhost:8080/, logging=true, sample=100%)
appFlavor: null

$ flutter test test/cfg --dart-define-from-file=config/prod.json
AppConfig(prod, https://api.example.com/, logging=false, sample=10%)
appFlavor: null

$ flutter test test/cfg --dart-define-from-file=config/prod.json --dart-define=API_URL=http://insecure.example.com/
config error: Bad state: prod must use https, got http://insecure.example.com/
appFlavor: null

$ flutter test test/cfg --dart-define=ENV=qa --dart-define=API_URL=https://x.example.com/
config error: Invalid argument (ENV): must be one of (dev, staging, prod): "qa"
appFlavor: null
```

Observed behaviour worth remembering:

- With no defines at all, the app refuses to start with a message that says how to fix it.
- JSON values keep their types (`false`, `10`) and arrive through `bool.fromEnvironment`/`int.fromEnvironment`.
- An individual `--dart-define` **overrode** the same key from the file — handy for one-off experiments — and the prod rule
  caught the insecure URL.
- `appFlavor` is `null` because no `--flavor` was given (and `flutter test` has no native build).

The same flags work for `flutter run` and `flutter build`. In IDEs, put them in the launch configuration (`args` in VS Code's
`launch.json`, "Additional run args" in Android Studio).

## Flavors: the native half

!!! warning "Not run here"
    Flavors configure native Android and iOS builds, which need a working Android toolchain and Xcode; on the machine used for this
    course, Android builds were blocked by unaccepted SDK licenses and Xcode wasn't installed, so the snippets below were not built. They follow the structure of the official Flutter flavors
    guides — check those guides for your Flutter version, as the Gradle and Xcode details change over time.

**Android** — in `android/app/build.gradle.kts`:

```kotlin
android {
    flavorDimensions += "env"
    productFlavors {
        create("dev") {
            dimension = "env"
            applicationIdSuffix = ".dev"      // installs next to prod
            resValue("string", "app_name", "MyApp Dev")
        }
        create("prod") {
            dimension = "env"
            resValue("string", "app_name", "MyApp")
        }
    }
}
```

and reference `@string/app_name` as the `android:label` in `AndroidManifest.xml`. Per-flavor resources (icons,
`google-services.json`) go in `android/app/src/dev/...` and `android/app/src/prod/...`.

**iOS/macOS** — in Xcode, duplicate build configurations per flavor (`Debug-dev`, `Release-dev`, …), create a scheme named
after each flavor that uses them, and set per-configuration build settings such as `PRODUCT_BUNDLE_IDENTIFIER` and the display
name.

Then build or run a flavor together with its defines:

```bash
flutter run --flavor dev --dart-define-from-file=config/dev.json
flutter build appbundle --flavor prod --dart-define-from-file=config/prod.json
flutter build ipa --flavor prod --dart-define-from-file=config/prod.json
```

At runtime, `appFlavor` (from `package:flutter/services.dart`) contains the flavor name the app was built with — in the SDK
it's simply another compile-time define, `FLUTTER_APP_FLAVOR`, set by the tool.

## Secrets: what *not* to put in defines

Anything compiled into the app — defines, assets, Dart code, native resources — can be extracted from the installed app by a
determined person. Defines are **configuration**, not a vault.

- Fine: API base URLs, public client ids, analytics *publishable* keys, feature switches.
- Not fine: server API secrets, database passwords, signing keys, private tokens. Those belong on a server you control; the app
  authenticates the *user* to your server, and your server calls third parties.
- `--obfuscate --split-debug-info=build/symbols` in release builds makes reverse engineering harder (and smaller), but it's not
  security.

Signing keys and store credentials live in CI secrets, not in the repo ([lesson 05](05-ci.md), [lesson 06](06-publishing.md)).

## How It Actually Works

`flutter run`/`build` passes each define to the Dart compiler as an environment declaration (the tool base64-encodes the list
and forwards it; `--dart-define-from-file` reads the JSON and adds each key). When the compiler sees
`const String.fromEnvironment('API_URL')`, it replaces the expression with the literal value during constant evaluation, so
the release binary contains `"https://api.example.com/"` where your code had the lookup. Tree shaking then removes branches
on constant conditions. `--flavor` selects the Gradle product flavor or Xcode scheme for the native build and also adds the
`FLUTTER_APP_FLAVOR` define, which is how `appFlavor` knows. Because all of this happens at compile time, changing a define
requires a rebuild — hot reload keeps the old values.

## Common mistakes

- **Non-const `fromEnvironment`** (`final url = String.fromEnvironment(...)` inside a function called at runtime, not in a
  const context): on some platforms it silently returns the default. Always `const`.
- **Expecting hot reload to pick up new defines.** Restart the build.
- **Secrets in defines or assets.**
- **Silent defaults for critical values** (`defaultValue: 'https://api.example.com/'` for `API_URL` in dev builds means a dev
  build can hit production). Prefer failing when unset.
- **Flavor and defines out of sync** (`--flavor prod` with `config/dev.json`). Wrap both in a script or CI job.

## Exercise

1. Add `config/staging.json` and a `FEATURE_NEW_CHECKOUT` boolean. Use it to choose between two checkout pages, and verify with a
   test run under both values.
2. Add a check that `appFlavor` (when not null) matches `ENV`, and fail at startup if they disagree.
3. Write a `scripts/build.sh dev|staging|prod` that runs `flutter build appbundle` with the matching flavor and defines file.
4. Build a release web app with and without `--dart-define=LOGGING=false` and compare the size of `main.dart.js`. Did the logging
   code get tree-shaken?
