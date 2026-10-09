---
description: "Call native Android and iOS code from Flutter with MethodChannel and EventChannel: API design, Kotlin and Swift handlers, codecs and mocks."
---

# 05 · Platform Channels

Flutter draws its own UI, but battery level, sensors, Bluetooth, the keychain, a vendor SDK — those live in the host
platform's APIs. Most of the time a plugin from pub.dev already wraps what you need. When it doesn't, **platform
channels** let your Dart code send messages to Kotlin/Java on Android and Swift/Objective-C on iOS, and get replies.

!!! warning "What was and wasn't run for this lesson"
    The Dart side and its tests below were run with `flutter test`. The Kotlin and Swift handlers were **not** compiled or
    run: on this course's build machine, Android builds were blocked by unaccepted SDK licenses and Xcode isn't
    installed. They follow the standard channel APIs, but
    treat them as a starting point and verify them in your own project.

## Three kinds of channel

| Channel | Shape | Use for |
|---|---|---|
| `MethodChannel` | request → one reply (a `Future`) | "get battery level", "start scan", "save to keychain" |
| `EventChannel` | subscribe → stream of events | sensor readings, connectivity changes, download progress |
| `BasicMessageChannel` | raw messages either way | custom protocols, rarely needed directly |

Every channel has a **name** (conventionally reverse-domain, like `com.example.app/device`) that both sides must agree on,
and a **codec** that turns values into bytes.

## The Dart side: a small, typed API

```dart title="lib/device_info.dart"
import 'package:flutter/services.dart';

/// Dart side of a small platform API.
class DeviceInfo {
  static const _methods = MethodChannel('com.example.app/device');
  static const _events = EventChannel('com.example.app/charging');

  /// Battery level 0–100, or null if the platform can't tell.
  static Future<int?> batteryLevel() async {
    try {
      return await _methods.invokeMethod<int>('getBatteryLevel');
    } on PlatformException catch (e) {
      // Native code reported an error (e.g. no battery on a desktop).
      if (e.code == 'UNAVAILABLE') return null;
      rethrow;
    } on MissingPluginException {
      // No native handler registered on this platform (e.g. web, or a missing implementation).
      return null;
    }
  }

  static Future<void> setKeepScreenOn(bool on) =>
      _methods.invokeMethod<void>('setKeepScreenOn', {'on': on});

  /// Stream of charging states: 'charging' | 'discharging' | 'full'.
  static Stream<String> chargingState() => _events.receiveBroadcastStream().cast<String>();
}
```

The rest of the app calls `DeviceInfo.batteryLevel()` and never sees a channel. Note the three failure modes handled:

- **`PlatformException`** — native code ran and reported an error with a code you defined (`UNAVAILABLE`).
- **`MissingPluginException`** — nothing on the other side is listening on that channel name. This happens on platforms
  you didn't implement (the web, desktop) and after adding native code without fully restarting the app (hot reload
  doesn't recompile native code).
- **Anything else** propagates, so real bugs aren't silently swallowed.

## The Android side (Kotlin)

```kotlin title="android/app/src/main/kotlin/.../MainActivity.kt"
import android.content.Context
import android.os.BatteryManager
import android.view.WindowManager
import io.flutter.embedding.android.FlutterActivity
import io.flutter.embedding.engine.FlutterEngine
import io.flutter.plugin.common.MethodChannel

class MainActivity : FlutterActivity() {
    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        MethodChannel(flutterEngine.dartExecutor.binaryMessenger, "com.example.app/device")
            .setMethodCallHandler { call, result ->
                when (call.method) {
                    "getBatteryLevel" -> {
                        val bm = getSystemService(Context.BATTERY_SERVICE) as BatteryManager
                        val level = bm.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)
                        if (level in 0..100) result.success(level)
                        else result.error("UNAVAILABLE", "Battery level not available", null)
                    }
                    "setKeepScreenOn" -> {
                        val on = call.argument<Boolean>("on") ?: false
                        if (on) window.addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON)
                        else window.clearFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON)
                        result.success(null)
                    }
                    else -> result.notImplemented() // becomes MissingPluginException in Dart
                }
            }
    }
}
```

Handlers run on the platform's **main thread**. Calling `result.success` exactly once is required; for slow work
(file I/O, network), move it to a background thread and post the result back to the main thread.

## The iOS side (Swift)

```swift title="ios/Runner/AppDelegate.swift (excerpt)"
let registrar = self.registrar(forPlugin: "DeviceInfo")!
let channel = FlutterMethodChannel(name: "com.example.app/device",
                                   binaryMessenger: registrar.messenger())
channel.setMethodCallHandler { call, result in
  switch call.method {
  case "getBatteryLevel":
    UIDevice.current.isBatteryMonitoringEnabled = true
    let level = UIDevice.current.batteryLevel // -1.0 when unknown, e.g. in the Simulator
    if level < 0 {
      result(FlutterError(code: "UNAVAILABLE", message: "Battery level not available", details: nil))
    } else {
      result(Int(level * 100))
    }
  case "setKeepScreenOn":
    let on = (call.arguments as? [String: Any])?["on"] as? Bool ?? false
    UIApplication.shared.isIdleTimerDisabled = on
    result(nil)
  default:
    result(FlutterMethodNotImplemented)
  }
}
```

Where this registration goes has changed across Flutter's iOS app templates (newer templates adopt the UIScene life cycle
and initialize the engine differently). Put it wherever your generated `ios/Runner` files obtain a plugin registrar or
binary messenger — check the template your project was created with rather than copying an old snippet verbatim.

## Testing the Dart side

You can't run Kotlin or Swift in `flutter test`, but you can replace the other end of the channel:

```dart title="test/device_info_test.dart"
import 'package:flutter/services.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:l2/a/device_info.dart';

void main() {
  TestWidgetsFlutterBinding.ensureInitialized();
  final messenger = TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger;
  const channel = MethodChannel('com.example.app/device');
  tearDown(() => messenger.setMockMethodCallHandler(channel, null));

  test('success, error, and missing implementation', () async {
    final calls = <String>[];
    messenger.setMockMethodCallHandler(channel, (call) async {
      calls.add('${call.method}(${call.arguments})');
      return switch (call.method) {
        'getBatteryLevel' => 87,
        'setKeepScreenOn' => null,
        _ => throw MissingPluginException(),
      };
    });
    print('battery: ${await DeviceInfo.batteryLevel()}');
    await DeviceInfo.setKeepScreenOn(true);
    print('calls seen by "native": $calls');

    messenger.setMockMethodCallHandler(channel, (call) async =>
        throw PlatformException(code: 'UNAVAILABLE', message: 'No battery'));
    print('desktop without battery: ${await DeviceInfo.batteryLevel()}');

    messenger.setMockMethodCallHandler(channel, null); // nothing registered at all
    print('no handler: ${await DeviceInfo.batteryLevel()}');

    messenger.setMockMethodCallHandler(channel, (call) async =>
        throw PlatformException(code: 'PERMISSION_DENIED', message: 'nope'));
    try {
      await DeviceInfo.batteryLevel();
    } on PlatformException catch (e) {
      print('other errors propagate: ${e.code} ${e.message}');
    }
  });

  test('what actually crosses the channel: bytes', () {
    const codec = StandardMethodCodec();
    final bytes = codec.encodeMethodCall(const MethodCall('setKeepScreenOn', {'on': true}));
    print('encoded call is ${bytes.lengthInBytes} bytes: ${bytes.buffer.asUint8List(bytes.offsetInBytes, bytes.lengthInBytes)}');
    final back = codec.decodeMethodCall(bytes);
    print('decoded: ${back.method} ${back.arguments}');
  });

  test('event channel stream', () async {
    const events = EventChannel('com.example.app/charging');
    messenger.setMockStreamHandler(events, MockStreamHandler.inline(onListen: (args, sink) {
      sink.success('discharging');
      sink.success('charging');
      sink.success('full');
      sink.endOfStream();
    }));
    print('events: ${await DeviceInfo.chargingState().toList()}');
  });
}
```

```console
$ flutter test test/device_info_test.dart
battery: 87
calls seen by "native": [getBatteryLevel(null), setKeepScreenOn({on: true})]
desktop without battery: null
no handler: null
other errors propagate: PERMISSION_DENIED nope
encoded call is 24 bytes: [7, 15, 115, 101, 116, 75, 101, 101, 112, 83, 99, 114, 101, 101, 110, 79, 110, 13, 1, 7, 2, 111, 110, 1]
decoded: setKeepScreenOn {on: true}
events: [discharging, charging, full]
00:00 +3: All tests passed!
```

The mock handler plays the native side, so every branch of `DeviceInfo` was exercised: a value, the `UNAVAILABLE` error
mapped to `null`, a channel with **no** handler (which throws `MissingPluginException` — also mapped to `null`), and an
unexpected error that propagated. The event channel test streamed three states and an end-of-stream.

The byte dump shows what really crosses the boundary. Decoding by hand with the standard codec's type tags:
`7` (string) `15` (length) then the 15 UTF-8 bytes of `setKeepScreenOn`; then `13` (map) `1` (one entry), `7 2 o n`
(the key string "on"), and `1` (true). The native side decodes the same bytes into a Kotlin `Map` or a Swift
`[String: Any]`.

## Supported types and Pigeon

`StandardMessageCodec` handles `null`, `bool`, `int`, `double`, `String`, `Uint8List` and other typed lists, `List` and
`Map` (of those types). Anything else must be converted — usually to a `Map`. Method names and argument keys are strings,
so a typo on either side is a runtime error.

**Pigeon** (`package:pigeon`) fixes that: you describe the API once as Dart abstract classes, and it generates matching
Dart, Kotlin, Swift (and other) code with typed classes and a custom codec. For anything beyond a couple of methods, it's
worth it. For calling C libraries directly (no Kotlin/Swift layer), see `dart:ffi` in [Level 4 · 08](../level-4/08-plugins-ffi.md).

## How It Actually Works

A channel is a thin convenience over the engine's **binary messenger**. `invokeMethod` encodes a `MethodCall` with the
codec into a `ByteData`, then calls `BinaryMessenger.send(channelName, bytes)`. The engine passes the bytes from the Dart
UI thread to the platform thread, where the embedder (the Android or iOS host code) looks up the handler registered for that
name and calls it. The handler's reply is encoded into an *envelope* — success with a value, or error with code, message
and details — and sent back; the Dart `Future` completes with the decoded value or throws `PlatformException`. If no handler
is registered, the reply is `null`, which `MethodChannel` turns into `MissingPluginException`.

Messages are asynchronous and ordered per channel. Because the platform handler runs on the main thread, a slow handler
blocks native UI work, and a Dart caller awaiting it simply waits — Flutter's UI doesn't freeze, but the result is late.
`EventChannel` uses the same messenger: listening sends a `listen` method call, the native side's `StreamHandler` starts
emitting success/error envelopes on the channel, and cancelling sends `cancel`.

## Common mistakes

- **Mismatched channel names or method names** between Dart and native. Keep them in constants; better, generate them
  with Pigeon.
- **Hot reload after editing native code** → `MissingPluginException`. Stop and rebuild.
- **Calling `result` twice, or never.** Never: the Dart future hangs forever. Twice: crash.
- **Blocking the platform main thread** with I/O inside the handler.
- **Leaking channel types into the UI.** Wrap channels in a Dart class with a typed API, as above.
- **No fallback for platforms you didn't implement** (web/desktop).

## Exercise

1. Add `getDeviceModel()` returning a `String`, with a Dart test for success and `notImplemented`.
2. Implement the `EventChannel` for charging state on Android using a `BroadcastReceiver` for
   `Intent.ACTION_BATTERY_CHANGED`. Remember to unregister it in `onCancel`.
3. Rewrite the API with Pigeon. Compare the generated Dart with `DeviceInfo` above.
4. Write a widget that shows the battery level and refreshes it every minute; test it with the mock handler and fake time.
