---
description: "Package native code for Flutter: plugin structure, platform interfaces, federated plugins, and calling C with dart:ffi — compiled and run."
---

# 08 · Plugins & dart:ffi

[Level 3 · 05](../level-3/05-platform-channels.md) wired a platform channel directly into an app. When native code should be
reused across apps — or published for others — it goes in a **plugin** package. And when the native code is C/C++ (or Rust, Go,
anything with a C ABI), you can skip the Kotlin/Swift layer entirely and call it from Dart with **`dart:ffi`**. This lesson
generates a plugin to study its structure, then builds a small C library and calls it through FFI, with real output.

## Anatomy of a plugin

```console
$ flutter create --template=plugin --platforms=android,ios --org com.example battery_info
```

The interesting files it generated (the `example/` app is omitted):

```text
lib/battery_info.dart                       public API the app imports
lib/battery_info_platform_interface.dart    abstract contract every platform implements
lib/battery_info_method_channel.dart        default implementation using a MethodChannel
android/build.gradle.kts
android/src/main/kotlin/com/example/battery_info/BatteryInfoPlugin.kt
android/src/test/kotlin/com/example/battery_info/BatteryInfoPluginTest.kt
ios/battery_info.podspec                    CocoaPods integration
ios/battery_info/Package.swift              Swift Package Manager integration
ios/battery_info/Sources/battery_info/BatteryInfoPlugin.swift
test/battery_info_test.dart
test/battery_info_method_channel_test.dart
```

and this in its `pubspec.yaml`, which tells the Flutter tool which native class to register on each platform:

```yaml
flutter:
  plugin:
    platforms:
      android:
        package: com.example.battery_info
        pluginClass: BatteryInfoPlugin
      ios:
        pluginClass: BatteryInfoPlugin
```

The generated Dart tests passed out of the box (`flutter test` → `00:00 +3: All tests passed!`); the Kotlin and Swift parts
weren't built (on this machine Android builds were blocked by unaccepted SDK licenses, and Xcode isn't installed).

### The platform interface pattern

```dart title="lib/battery_info_platform_interface.dart (generated)"
import 'package:plugin_platform_interface/plugin_platform_interface.dart';

import 'battery_info_method_channel.dart';

abstract class BatteryInfoPlatform extends PlatformInterface {
  BatteryInfoPlatform() : super(token: _token);

  static final Object _token = Object();

  static BatteryInfoPlatform _instance = MethodChannelBatteryInfo();

  static BatteryInfoPlatform get instance => _instance;

  static set instance(BatteryInfoPlatform instance) {
    PlatformInterface.verifyToken(instance, _token);
    _instance = instance;
  }

  Future<String?> getPlatformVersion() {
    throw UnimplementedError('platformVersion() has not been implemented.');
  }
}
```

The app calls the public API, which delegates to `BatteryInfoPlatform.instance`. The default instance talks over a method
channel, but any platform can replace it — a web implementation written in Dart with `package:web`, a Windows implementation
using FFI, or a fake in tests. The `token` check (from `plugin_platform_interface`) makes sure implementations *extend* the
interface rather than `implements` it, so adding a new method with a default body later isn't a breaking change for every
implementation.

### Federated plugins

Large plugins split this across packages: `battery_info` (app-facing), `battery_info_platform_interface`, and one package per
platform (`battery_info_android`, `battery_info_ios`, `battery_info_web`…), each declaring itself the *endorsed* implementation
for its platform. Different teams can own different platforms, and a community member can add, say, Linux support without
touching the main package. Most first-party plugins (`shared_preferences`, `url_launcher`, `path_provider`) are structured this
way, which is why their dependency lists include several `_android`/`_ios`/`_web` packages.

## dart:ffi: calling C directly

FFI ("foreign function interface") lets Dart call functions in a native library and work with native memory, with no message
passing and no codec — calls are synchronous and fast. The C side:

```c title="native/checksum.c"
#include <stdint.h>
#include <stddef.h>

// Adler-32 checksum over a byte buffer (the algorithm zlib uses).
uint32_t adler32(const uint8_t* data, size_t len) {
  uint32_t a = 1, b = 0;
  const uint32_t MOD = 65521;
  for (size_t i = 0; i < len; i++) {
    a = (a + data[i]) % MOD;
    b = (b + a) % MOD;
  }
  return (b << 16) | a;
}

// Fills `out` with the count of each byte value; returns the number of distinct values.
int histogram(const uint8_t* data, size_t len, uint32_t out[256]) {
  for (int i = 0; i < 256; i++) out[i] = 0;
  for (size_t i = 0; i < len; i++) out[data[i]]++;
  int distinct = 0;
  for (int i = 0; i < 256; i++) if (out[i]) distinct++;
  return distinct;
}
```

```console
$ clang -O2 -shared -fPIC -o native/libchecksum.dylib native/checksum.c
```

The Dart side, using `package:ffi` (2.2.0 here) for its memory helpers:

```dart title="bin/main.dart"
import 'dart:convert';
import 'dart:ffi';
import 'dart:io';
import 'dart:typed_data';
import 'package:ffi/ffi.dart';

// 1. Describe the C signatures (native types) and the Dart signatures (Dart types).
typedef Adler32C = Uint32 Function(Pointer<Uint8> data, Size len);
typedef Adler32Dart = int Function(Pointer<Uint8> data, int len);
typedef HistogramC = Int32 Function(Pointer<Uint8> data, Size len, Pointer<Uint32> out);
typedef HistogramDart = int Function(Pointer<Uint8> data, int len, Pointer<Uint32> out);

class Checksum {
  Checksum(String path) : _lib = DynamicLibrary.open(path) {
    _adler32 = _lib.lookupFunction<Adler32C, Adler32Dart>('adler32');
    _histogram = _lib.lookupFunction<HistogramC, HistogramDart>('histogram');
  }
  final DynamicLibrary _lib;
  late final Adler32Dart _adler32;
  late final HistogramDart _histogram;

  /// Copies bytes into native memory, calls C, frees the memory.
  int adler32(Uint8List bytes) => using((arena) {
        final ptr = arena<Uint8>(bytes.length);
        ptr.asTypedList(bytes.length).setAll(0, bytes);
        return _adler32(ptr, bytes.length);
      });

  ({int distinct, int mostCommonByte}) histogram(Uint8List bytes) => using((arena) {
        final data = arena<Uint8>(bytes.length)..asTypedList(bytes.length).setAll(0, bytes);
        final out = arena<Uint32>(256);
        final distinct = _histogram(data, bytes.length, out);
        final counts = out.asTypedList(256);
        var best = 0;
        for (var i = 1; i < 256; i++) {
          if (counts[i] > counts[best]) best = i;
        }
        return (distinct: distinct, mostCommonByte: best);
      });
}

/// The same algorithm in Dart, to check the C result and compare speed.
int adler32Dart(Uint8List data) {
  var a = 1, b = 0;
  for (final x in data) {
    a = (a + x) % 65521;
    b = (b + a) % 65521;
  }
  return (b << 16) | a;
}

void main() {
  final lib = Checksum('native/libchecksum.dylib');
  final wiki = Uint8List.fromList(utf8.encode('Wikipedia'));
  print('adler32("Wikipedia") C=0x${lib.adler32(wiki).toRadixString(16)} Dart=0x${adler32Dart(wiki).toRadixString(16)}');
  final h = lib.histogram(Uint8List.fromList(utf8.encode('mississippi')));
  print('histogram("mississippi"): ${h.distinct} distinct bytes, most common "${String.fromCharCode(h.mostCommonByte)}"');

  final big = Uint8List.fromList(List.generate(50 * 1024 * 1024, (i) => (i * 31) & 0xff));
  for (final (name, f) in [('C via FFI', lib.adler32), ('pure Dart', adler32Dart)]) {
    final sw = Stopwatch()..start();
    final r = f(big);
    print('${name.padRight(10)} 50 MB: 0x${r.toRadixString(16)} in ${sw.elapsedMilliseconds} ms');
  }
  print('running on ${Platform.operatingSystem} ${Abi.current()}');
}
```

```console
$ dart run bin/main.dart
adler32("Wikipedia") C=0x11e60398 Dart=0x11e60398
histogram("mississippi"): 4 distinct bytes, most common "i"
C via FFI  50 MB: 0xdf5159ea in 169 ms
pure Dart  50 MB: 0xdf5159ea in 249 ms
running on macos macos_arm64

$ dart compile exe bin/main.dart -o ffi_demo && ./ffi_demo
adler32("Wikipedia") C=0x11e60398 Dart=0x11e60398
histogram("mississippi"): 4 distinct bytes, most common "i"
C via FFI  50 MB: 0xdf5159ea in 171 ms
pure Dart  50 MB: 0xdf5159ea in 234 ms
running on macos macos_arm64
```

- **Correctness first**: both implementations agree, and `0x11E60398` is the well-known Adler-32 of "Wikipedia". "mississippi"
  has four distinct letters; `i` and `s` both appear four times, and the loop keeps the first maximum it finds, `i`.
- **Speed**: C was faster here (~170 ms vs ~235–250 ms for 50 MB, on an Apple Silicon Mac), and that *includes* copying 50 MB into
  native memory before the call. AOT-compiled Dart (`dart compile exe`, which is how Flutter release builds work) narrowed the gap
  slightly. That's the honest picture of FFI performance: Dart is already compiled to native code, so FFI pays off for
  existing, heavily optimized native libraries (SQLite, image codecs, crypto, ML runtimes) far more than for rewriting a loop in C.

Memory rules the example follows:

- Native memory isn't garbage collected. `using((arena) { ... })` allocates from an **arena** and frees everything when the block
  ends — even if it throws. Without it, you `malloc`/`calloc` and must call `free` yourself.
- `ptr.asTypedList(n)` views native memory as a Dart list *without copying* — fast, but the view is invalid once the memory is freed.
  Don't let it escape the `using` block.
- Types must match the C signature exactly (`Size` for `size_t`, `Uint32` for `uint32_t`). A mismatch isn't caught by the compiler —
  it's undefined behaviour at runtime.

## FFI in a Flutter app

- **Bundling the library**: `flutter create --template=plugin_ffi` scaffolds a package whose C sources are compiled for each platform
  by the platform build systems. Newer Dart and Flutter releases also offer **build hooks** ("native assets") that compile and bundle
  native code from a `hook/build.dart` script; check the current docs for its status on your version.
- **Loading**: `DynamicLibrary.open('libname.so')` on Android/Linux, `.dll` on Windows; on iOS/macOS libraries linked into the app
  are often found with `DynamicLibrary.process()`.
- **Generating bindings**: `package:ffigen` reads C headers and generates the `typedef`s and lookups above, which avoids hand-typing
  signatures (and getting them subtly wrong).
- **Threads**: FFI calls run on the calling isolate's thread. A slow C function blocks the UI just like slow Dart code — call it from
  a background isolate ([Level 3 · 06](../level-3/06-isolates.md)).
- **Web**: there's no `dart:ffi` on the web; use `dart:js_interop` to call JavaScript, or compile the C library to Wasm separately.

## How It Actually Works

`DynamicLibrary.open` asks the OS loader (`dlopen` on macOS/Linux/Android, `LoadLibrary` on Windows) to map the shared library into
the process. `lookupFunction` finds the exported symbol's address and has the Dart VM generate a **trampoline**: a small piece of
machine code that converts Dart values to the C calling convention for the platform's ABI (which registers hold which arguments),
switches from Dart's stack and execution state to native, calls the function, and converts the return value back. Because Dart and C
share one process and address space, a `Pointer<Uint8>` is literally a memory address — that's what makes FFI fast, and also why
a wrong signature or a dangling pointer can crash the whole app instead of throwing a Dart exception.

Method channels ([Level 3 · 05](../level-3/05-platform-channels.md)) are the opposite trade-off: asynchronous messages, values
serialized through a codec, and native code running on the platform thread in Kotlin or Swift — slower per call, but safe, and the
right tool for platform APIs that only exist in Java/Kotlin/Objective-C/Swift.

## Common mistakes

- **Leaking native memory** — allocate in an arena or pair every `malloc` with `free`.
- **Keeping `asTypedList` views** after the memory is freed.
- **Mismatched types** between the `typedef` and the C signature (e.g. `Int32` for a `size_t`).
- **Expecting C to be dramatically faster** than AOT Dart for simple loops; measure.
- **Breaking plugin implementations** by adding abstract methods to a platform interface; add methods with default bodies.
- **Long-running FFI calls on the UI isolate.**

## Exercise

1. Add `int crc32(const uint8_t*, size_t)` to the C library, bind it, and compare it against `package:archive`'s Dart CRC32 for
   correctness and speed.
2. Remove `using` and allocate with `malloc` without freeing, in a loop of 10,000 calls. Watch the process memory (Activity Monitor
   or `top`). Then fix it.
3. Generate the bindings with `ffigen` from a `checksum.h` header and diff them against the hand-written ones.
4. Implement `getBatteryLevel` in the generated plugin for Android and iOS using the code from Level 3 · 05, and add a web
   implementation that uses the browser's Battery Status API where available (it isn't in every browser).
