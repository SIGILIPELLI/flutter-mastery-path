---
description: "Keep a Flutter app healthy for years: version constraints and lockfiles, pub outdated, dart fix, SDK pinning and a safe upgrade routine."
---

# 09 · Upgrades, Dependencies & Long-Lived Apps

Shipping version 1.0 is the start. Flutter ships a new stable release every few months, Dart evolves alongside it, packages
release breaking majors, and the app stores raise minimum SDK requirements on their own schedule. Apps that upgrade a little,
often, stay cheap to maintain; apps that skip a year face a painful big-bang migration. This lesson is about the routine — run
here against this course's own example project.

## Version constraints and the lockfile

```yaml title="pubspec.yaml (excerpt)"
environment:
  sdk: ^3.12.2

dependencies:
  go_router: ^18.0.2
  flutter_riverpod: ^3.4.3
  http: ^1.6.0
```

- `^18.0.2` means `>=18.0.2 <19.0.0`: any compatible version under semantic versioning. For `0.x` packages, the caret is
  narrower — `^0.20.2` means `<0.21.0` — because minor versions are allowed to break before 1.0.
- `pubspec.lock` records the **exact** versions resolved. **Commit it for apps** (so every developer and CI build uses the same
  code) and don't for packages you publish (so you're tested against a range).
- `flutter pub get` respects the lockfile; `flutter pub upgrade` moves to the newest versions allowed by your constraints;
  `flutter pub upgrade --major-versions` *rewrites the constraints* to the latest majors — the one that can break things.

## What's out of date?

```console
$ flutter pub outdated
Showing outdated packages.
[*] indicates versions that are not the latest available.

Package Name              Current   Upgradable  Resolvable  Latest

direct dependencies:
cupertino_icons           *1.0.9    *1.0.9      2.0.0       2.0.0
intl                      *0.20.2   *0.20.2     *0.20.2     0.20.3

dev_dependencies: all up-to-date.

transitive dependencies:
clock                     *1.1.2    *1.1.2      *1.1.2      1.1.3
matcher                   *0.12.19  *0.12.19    *0.12.19    0.12.20
meta                      *1.18.0   *1.18.0     *1.18.0     1.19.0
...
1 dependency is constrained to a version that is older than a resolvable version.
To update it, edit pubspec.yaml, or run `flutter pub upgrade --major-versions`.
```

How to read the columns:

- **Upgradable**: what `pub upgrade` would give you within your constraints.
- **Resolvable**: the newest version that *could* work with everything else if you changed your constraint. `cupertino_icons`
  2.0.0 is resolvable — a major bump, so read its changelog before taking it.
- **Latest**: the newest published. When *Resolvable* is older than *Latest* (as for `intl` and the transitive packages here),
  something else is holding it back. In a Flutter app that's very often the **SDK itself**: `flutter`, `flutter_test` and
  `flutter_localizations` pin exact versions of packages such as `intl`, `meta` and `matcher`, and those move only when you upgrade
  Flutter. Don't fight these with `dependency_overrides`.

## Let the tools migrate for you: `dart fix`

Many deprecations and lint fixes come with automated fixes. Always look first:

```console
$ dart fix --dry-run
Computing fixes in l2 (dry run)...

16 proposed fixes in 11 files.

lib/a/element_demo.dart
  curly_braces_in_flow_control_structures - 1 fix
...
lib/m/render_demo.dart
  prefer_initializing_formals - 4 fixes
...
lib/s/api_client.dart
  use_null_aware_elements - 1 fix
```

then apply, re-format, and prove nothing broke:

```console
$ dart fix --apply
16 fixes made in 11 files.
$ dart format lib test
Formatted 64 files (1 changed) in 0.10 seconds.
$ flutter test
00:08 +99: All tests passed!
```

After that, the analyzer's only remaining finding outside the intentional `print`s in tests was a naming lint
(`Row_` isn't UpperCamelCase) — which needs a human decision, so `dart fix` correctly left it alone. When Flutter deprecates an
API, the framework typically ships a data-driven fix with it, so `dart fix` after an SDK upgrade rewrites most call sites for you
(for example, the course's code uses `withValues(alpha: ...)` rather than the deprecated `withOpacity`).

## Upgrading Flutter itself

```console
$ flutter --version
Flutter 3.44.8 • channel stable • https://github.com/flutter/flutter.git
$ flutter channel
Flutter channels:
  master (latest development branch, for contributors)
  main (latest development branch, follows master channel)
  beta (updated monthly, recommended for experienced users)
  ...
```

- **Use `stable`** for apps. `beta` is for trying upcoming features; `main`/`master` for contributors.
- **Pin the version per project** so everyone and CI use the same SDK: tools like FVM (`.fvmrc`), asdf/mise, or a documented
  version plus the `environment: flutter: ">=x.y.z"` constraint and CI's `flutter-version` ([lesson 05](05-ci.md)). Otherwise "works on
  my machine" returns with different SDKs.
- **Read the release notes and the breaking-changes page** for every version you cross. Flutter publishes migration guides for
  each breaking change.

## A safe upgrade routine

1. Start from a green `main` with a passing CI.
2. Branch. Upgrade **one thing at a time**: the Flutter SDK, *or* one major package — not everything at once.
3. `flutter pub get` → `dart fix --dry-run` → `dart fix --apply` → `flutter analyze`.
4. Read changelogs for anything with a major bump; search your code for the APIs they mention.
5. Run unit, widget and golden tests. Expect golden diffs after SDK upgrades (rendering changes); inspect them before
   regenerating.
6. Build every target platform in CI (Android Gradle/Kotlin and iOS CocoaPods/SPM changes often come with SDK upgrades).
7. Run integration tests and a manual smoke test on real devices.
8. Merge, release to internal testing, then roll out.

A small upgrade each month is usually an hour's work. Twelve at once is a project.

## Platform requirements move too

- **Google Play** periodically raises the required `targetSdkVersion` for new apps and updates; Flutter's templates track this, but
  your `android/` folder only changes when you change it.
- **Apple** periodically requires building with a recent Xcode/iOS SDK for App Store submissions.
- **Native tooling** — Gradle, the Android Gradle Plugin, Kotlin, CocoaPods vs Swift Package Manager — changes with Flutter
  releases. When a new Flutter version's template differs from your project, compare with a freshly generated
  `flutter create` project of the same version, and port the differences.

Check the current requirements in the Play Console and App Store Connect documentation rather than relying on dates in
tutorials (including this one).

## Choosing dependencies you'll be able to upgrade

Before adding a package, check on pub.dev: publisher (verified?), recent releases, open issues, null safety and Dart 3
compatibility, platform support, the pub points score, and whether it's a thin wrapper you could write yourself in 50 lines.
Every dependency is a future upgrade. Prefer packages from the Dart and Flutter teams or well-maintained publishers for
foundational needs (routing, storage, HTTP), and isolate any risky dependency behind your own interface
([lesson 03](03-architecture.md)) so replacing it is a local change.

## How It Actually Works

`pub` resolves versions with a solver (PubGrub) that searches for one version of every package satisfying all constraints in the
graph — yours, your dependencies', and the SDK's. When no solution exists, it explains the conflict as a chain of incompatibilities
("because every version of X depends on Y ^2.0 and your app depends on Y ^1.0…"). The result is written to `pubspec.lock` and to
`.dart_tool/package_config.json`, which tells the compiler where each package's source lives in the pub cache. `dart fix` runs the
analyzer, collects diagnostics that have an associated *fix*, and applies them — including data-driven fixes that packages
(including the Flutter framework) declare in `fix_data.yaml` files to describe renames and API changes, which is how an SDK
upgrade can come with its own migration.

## Common mistakes

- **Never upgrading**, then needing to jump several majors under deadline pressure.
- **Upgrading everything at once**, so when something breaks nobody knows which change caused it.
- **Not committing `pubspec.lock`** in an app.
- **`dependency_overrides` left in place** after a temporary workaround.
- **Regenerating goldens blindly** after an upgrade.
- **Unpinned SDK in CI.**

## Exercise

1. Run `flutter pub outdated` on one of your projects and classify each row: safe minor upgrade, major needing a changelog read, or
   held back by the SDK.
2. Upgrade `cupertino_icons` to 2.x in a branch following the routine above. Did anything change?
3. Add FVM (or your team's version manager) to a project and make CI read the same version.
4. Write a `docs/UPGRADING.md` for your team with the routine, who owns it, and how often it runs.
