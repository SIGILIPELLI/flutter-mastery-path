---
description: "Ship a Flutter app: versioning, icons, Android upload keys and App Bundles, iOS signing and TestFlight, privacy declarations and rollouts."
---

# 06 · Building & Publishing to the Stores

Getting an app from `flutter run` to an install button involves less code and more process than anything else in this course:
accounts, keys, metadata, reviews. This lesson walks through the pipeline for Google Play and the Apple App Store in the order
you'll meet it, with the parts that can bite you later called out.

!!! warning "What was and wasn't run"
    Store submission requires paid developer accounts, and Android/iOS release builds need a working Android toolchain and Xcode.
    On the machine used for this course, the Android SDK's licenses had not been accepted (so Android builds couldn't run), Xcode
    wasn't installed, and no store accounts were used. The `keytool` and web build commands below were run and their
    output is real; Android/iOS build and upload steps are described from the documented process and were **not** executed here.
    Store policies, console screens and requirements (target API levels, SDK minimums, privacy forms) change frequently;
    always check the current Play Console and App Store Connect documentation before a release.

## 1. Versioning

`pubspec.yaml` holds both version numbers:

```yaml
version: 1.4.0+27   # build-name (shown to users) + build-number (must increase every upload)
```

- **Build name** (`1.4.0`) becomes Android's `versionName` and iOS's `CFBundleShortVersionString`.
- **Build number** (`27`) becomes Android's `versionCode` and iOS's `CFBundleVersion`. Each store rejects an upload whose number
  isn't higher than the last one. CI usually injects it: `flutter build appbundle --build-number=$GITHUB_RUN_NUMBER`.

You can see the mapping on any platform; for the web target it lands in `version.json`:

```console
$ flutter build web --release -t lib/rl/main.dart --build-name=1.4.0 --build-number=27
...
✓ Built build/web
$ cat build/web/version.json
{"app_name":"l2","version":"1.4.0","build_number":"27","package_name":"l2"}
```

## 2. Identity, name and icons

- **Application id / bundle id** (`com.yourcompany.app`) is permanent once published. Choose it before the first upload. Set it
  with `flutter create --org com.yourcompany` or in `android/app/build.gradle(.kts)` (`applicationId`) and Xcode
  (`PRODUCT_BUNDLE_IDENTIFIER`).
- **Display name**: `android:label` in `AndroidManifest.xml`; `CFBundleDisplayName` in `ios/Runner/Info.plist`.
- **Icons**: Android needs adaptive icons (foreground + background layers) at several densities; iOS needs an asset catalog with
  a 1024×1024 marketing icon. The `flutter_launcher_icons` package generates them from one source image; check its output on
  real devices (adaptive-icon safe zones crop more than you'd expect).
- **Splash screen**: native launch screens are configured per platform; `flutter_native_splash` automates it.

## 3. Android: sign, build, upload

**Create an upload key** once and keep it safe (a password manager or your CI's secret store — never the repo). Here's a
throwaway one generated for this lesson, with the password supplied from an environment variable rather than typed on the
command line:

```console
$ keytool -genkeypair -v -keystore demo-upload.jks -storetype PKCS12 -keyalg RSA -keysize 2048 \
    -validity 10000 -alias upload -dname "CN=Example Dev, O=Example, C=US" -storepass:env KS_PASS
Generating 2,048 bit RSA key pair and self-signed certificate (SHA384withRSA) with a validity of 10,000 days
	for: CN=Example Dev, O=Example, C=US
[Storing demo-upload.jks]

$ keytool -list -keystore demo-upload.jks -storepass:env KS_PASS
Keystore type: PKCS12
Keystore provider: SUN
Your keystore contains 1 entry
upload, 09-Oct-2026, PrivateKeyEntry,
Certificate fingerprint (SHA-256): 82:98:76:B6:6E:3C:DC:07:9B:DE:45:8F:EF:34:8F:0C:A6:0B:94:18:FE:A6:05:70:95:D0:F2:4F:E1:93:E3:56
```

**Wire it into Gradle** without committing secrets: a `android/key.properties` file (git-ignored) or CI environment variables,
read in `android/app/build.gradle.kts`:

```kotlin
import java.util.Properties

val keyProps = Properties().apply {
    val f = rootProject.file("key.properties")
    if (f.exists()) f.inputStream().use { load(it) }
}

android {
    signingConfigs {
        create("release") {
            storeFile = keyProps["storeFile"]?.let { file(it as String) }
            storePassword = keyProps["storePassword"] as String?
            keyAlias = keyProps["keyAlias"] as String?
            keyPassword = keyProps["keyPassword"] as String?
        }
    }
    buildTypes {
        getByName("release") { signingConfig = signingConfigs.getByName("release") }
    }
}
```

**Build an App Bundle** (Play's required format for new apps):

```bash
flutter build appbundle --release --obfuscate --split-debug-info=build/symbols
```

**Play App Signing**: Google holds the key that signs what users install; your *upload key* only proves uploads come from you.
If you lose the upload key it can be reset through Play Console support — far better than losing an app signing key, which is
why Play App Signing is the default for new apps.

**Upload** to an **internal testing** track first (available to a small list of testers within minutes), then closed/open
testing, then production with a **staged rollout** (e.g. 5 % → 20 % → 100 %), watching crash and ANR rates in Android vitals at
each step. New personal developer accounts may face additional testing requirements before production access — read the
current Play Console rules for your account type.

## 4. iOS: sign, build, upload

- **Apple Developer Program** membership (paid, yearly) is required to distribute.
- **Signing**: in Xcode → Runner target → *Signing & Capabilities*, select your team and enable *Automatically manage signing*;
  Xcode creates the certificates and provisioning profiles. Teams with CI often use manual signing with `fastlane match` or
  App Store Connect API keys.
- **App Store Connect**: create the app record with the same bundle id.
- **Build and upload**:

```bash
flutter build ipa --release --obfuscate --split-debug-info=build/symbols
# then upload build/ios/ipa/*.ipa with Apple's Transporter app, Xcode's Organizer,
# or `xcrun altool`/the App Store Connect API from CI
```

- **TestFlight** distributes builds to internal testers immediately and to external testers after a light beta review.
- **App Review** checks the production submission against Apple's guidelines; plan for review time and possible rejections
  (common causes: crashes, incomplete metadata, login-walled apps without a demo account, unclear purpose of permissions).

## 5. Privacy declarations

Both stores require you to declare what data you collect and why — Play's **Data safety** form and Apple's **App Privacy**
details — and they must cover your third-party SDKs too (analytics, crash reporting, ads). Apple additionally requires
**privacy manifests** (`PrivacyInfo.xcprivacy`) declaring use of certain "required reason" APIs; Flutter and many plugins ship
their own manifests, but your app's declarations are your responsibility. Every permission prompt needs a usage string that
explains *why* (`NSCameraUsageDescription` etc. on iOS). Inaccurate declarations are a common cause of rejection and a real
trust problem.

## 6. Release hygiene

- **Obfuscation and symbols**: `--obfuscate --split-debug-info=<dir>` shrinks and obscures Dart symbols. Keep the symbol files for
  each release — you need them (`flutter symbolize`) to read stack traces from production crashes.
- **Crash reporting** (Firebase Crashlytics, Sentry, others) wired up before the first public release.
- **Store listing**: screenshots for required device sizes, short and full descriptions, a privacy-policy URL, content rating
  questionnaire, support contact.
- **Release checklist** in the repo, so it's the same every time — and automated in CI where possible (fastlane or the
  stores' APIs).

## How It Actually Works

`flutter build appbundle` compiles your Dart code ahead-of-time into native machine code (`libapp.so`) for each target ABI,
bundles it with the Flutter engine (`libflutter.so`), assets and the Android host app, and lets Gradle produce an `.aab`. Google
Play then generates optimized APKs per device configuration (ABI, screen density, language) from that bundle and signs them with
the app signing key — that's why users download less than the bundle size. On iOS, `flutter build ipa` AOT-compiles Dart into an
`App.framework`, Xcode archives the Runner app with `Flutter.framework`, signs it with your distribution certificate, and exports
an `.ipa`; Apple re-processes and thins it per device. In both cases the signature is what lets the OS verify that an update comes
from the same publisher as the installed version — which is why losing signing keys is the one mistake in this lesson that can't
be undone easily.

## Common mistakes

- **Committing keystores or `key.properties`.** Add them to `.gitignore` before creating them.
- **Reusing a build number**, rejected at upload.
- **Changing the application id after launch** — that's a new app to the store.
- **Debug-only behaviour in release** (logging secrets, test endpoints) — check `kReleaseMode` and your config.
- **Losing the split-debug-info symbols**, making production stack traces unreadable.
- **Skipping internal testing** and finding a release-only crash in production.

## Exercise

1. Write a `RELEASE.md` checklist for your app covering versioning, symbols, store listing, privacy forms and rollout steps.
2. Generate your own upload keystore with `keytool` (keep the password in a password manager), and write the `key.properties`
   + Gradle config. Confirm `git status` doesn't show either file.
3. Add a CI job that builds an App Bundle on tags, with the keystore restored from a base64-encoded CI secret.
4. Fill in a draft of the Play Data safety answers for the Level 3 habit tracker. What data does it collect, if any, once sync is
   connected to a real server?
