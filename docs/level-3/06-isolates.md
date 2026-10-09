---
description: "Keep Flutter's UI thread free: measure how heavy work blocks frames, move it to Isolate.run or compute, and understand message-copy costs."
---

# 06 · Isolates & Heavy Work

`async`/`await` does not make code run in parallel. It lets one thread *wait* without blocking — for a network reply, a
file, a timer. But CPU work — decoding a huge JSON response, resizing an image, searching a large list — runs on the same
thread as your UI, and while it runs, no frames are built. At 60 Hz a frame has about 16 ms; at 120 Hz about 8 ms. Anything
longer is a visible stutter ("jank").

Dart's answer is **isolates**: separate threads of execution, each with its own memory, that communicate by passing
messages. This lesson measures the problem and the fix.

## Measuring the problem

The script below builds a large JSON payload, then parses and summarises it twice — once on the main isolate, once in a
background isolate — while a 16 ms periodic timer stands in for frame rendering. If the main isolate is busy, the timer
can't fire.

```dart title="bin/isolate_demo.dart"
import 'dart:async';
import 'dart:convert';
import 'dart:isolate';

/// Builds a large JSON document (~ several MB) to stand in for a big API response.
String bigJson(int n) => jsonEncode([
      for (var i = 0; i < n; i++)
        {'id': i, 'name': 'Item $i', 'tags': ['a', 'b', 'c'], 'price': i * 1.25, 'meta': {'created': '2026-01-01', 'score': i % 97}}
    ]);

/// Pretend this is your app's work: decode, then summarise.
int parseAndSum(String json) {
  final list = jsonDecode(json) as List;
  var total = 0;
  for (final e in list) {
    total += (e as Map)['meta']['score'] as int;
  }
  return total;
}

/// Runs [work] while a 16 ms periodic timer tries to "draw frames" on this isolate.
Future<void> measure(String label, Future<int> Function() work) async {
  var ticks = 0;
  var longestGapMs = 0;
  var last = DateTime.now();
  final timer = Timer.periodic(const Duration(milliseconds: 16), (_) {
    final now = DateTime.now();
    final gap = now.difference(last).inMilliseconds;
    if (gap > longestGapMs) longestGapMs = gap;
    last = now;
    ticks++;
  });
  final sw = Stopwatch()..start();
  final result = await work();
  sw.stop();
  // Account for the gap between the last tick and the end of the work.
  final tail = DateTime.now().difference(last).inMilliseconds;
  if (tail > longestGapMs) longestGapMs = tail;
  timer.cancel();
  print('${label.padRight(22)} result=$result  took=${sw.elapsedMilliseconds} ms  '
      'frames ticked=$ticks  longest gap=$longestGapMs ms');
}

Future<void> main() async {
  final json = bigJson(200000);
  print('payload: ${(json.length / 1e6).toStringAsFixed(1)} MB of JSON');

  await measure('main isolate', () async => parseAndSum(json));
  await measure('Isolate.run', () => Isolate.run(() => parseAndSum(json)));

  // Cost of moving data: sending a big String copies it; returning a small int is cheap.
  final sw = Stopwatch()..start();
  await Isolate.run(() => 1);
  print('empty Isolate.run round trip: ${sw.elapsedMicroseconds} µs');
}
```

Three runs on an Apple Silicon Mac with `dart run` (Dart 3.12.2):

```console
$ dart run bin/isolate_demo.dart
payload: 23.0 MB of JSON
main isolate           result=9599419  took=245 ms  frames ticked=0  longest gap=246 ms
Isolate.run            result=9599419  took=229 ms  frames ticked=14  longest gap=26 ms
empty Isolate.run round trip: 279 µs

payload: 23.0 MB of JSON
main isolate           result=9599419  took=244 ms  frames ticked=0  longest gap=245 ms
Isolate.run            result=9599419  took=224 ms  frames ticked=13  longest gap=20 ms
empty Isolate.run round trip: 299 µs

payload: 23.0 MB of JSON
main isolate           result=9599419  took=244 ms  frames ticked=0  longest gap=245 ms
Isolate.run            result=9599419  took=222 ms  frames ticked=13  longest gap=21 ms
empty Isolate.run round trip: 288 µs
```

What the numbers say — for this machine and this payload; your device will differ, phones especially:

- On the main isolate, **zero** "frames" ticked during a ~245 ms parse. In an app, that's a quarter-second freeze: the
  spinner stops spinning, scrolling stops.
- With `Isolate.run`, the total time was about the same (the work didn't get faster), but the timer kept ticking, with the
  longest gap around 20–26 ms instead of 245 ms. The main isolate stayed responsive.
- Starting an isolate isn't free — a few hundred microseconds here for an empty task — so offloading tiny jobs costs more
  than it saves.

## The APIs

```dart
// Dart 2.19+: run a function in a new isolate and get its result.
final total = await Isolate.run(() => parseAndSum(json));

// Flutter's wrapper: same idea, top-level/static function + one argument.
final total = await compute(parseAndSum, json);
```

`compute` (from `package:flutter/foundation.dart`) predates `Isolate.run` and is still widely used. One practical
difference: **on the web there are no Dart isolates**, and `compute` falls back to running the function on the main thread
(asynchronously, but still blocking). For truly parallel web work you need web workers, which is a different topic.

For long-lived background work — a worker that receives many requests, or a stream of results — use `Isolate.spawn` with a
`ReceivePort`/`SendPort` pair, or a package that manages a pool of isolates.

## What can cross between isolates

Isolates don't share mutable memory. Arguments and results are **messages**: most objects are copied into the other
isolate's heap, while some immutable objects can be shared without copying. Practical rules:

- Plain data (numbers, strings, lists, maps, your own simple classes) can be sent.
- Things tied to the current isolate can't: open sockets, `ReceivePort`s you don't own, objects holding native resources.
  The error message names the offending object.
- Closures passed to `Isolate.run` capture variables — make sure they don't accidentally capture something huge or
  unsendable (like a `BuildContext`, or `this` of a widget). Prefer top-level or static functions with explicit arguments.
- Big inputs cost time to copy. Return **small results** (a sum, a parsed list of models, the bytes of a thumbnail) rather
  than intermediate data. `TransferableTypedData` moves large byte buffers without copying.

Plugins that talk to native code ([lesson 05](05-platform-channels.md)) generally must be called from the main isolate,
unless you set up a `BackgroundIsolateBinaryMessenger` for them.

## Do you actually need an isolate?

Check first, in **profile mode on a real device**, with DevTools' performance view ([Level 4 · 02](../level-4/02-performance.md)).
Common situations where yes:

- decoding JSON responses in the megabytes;
- image processing (decode, resize, filters) in Dart;
- cryptography, compression, parsing big files;
- searching/sorting tens of thousands of items on each keystroke.

And common situations where the cause is something else: rebuilding too much, layout of huge non-lazy lists (use
`ListView.builder`), expensive `paint`, or synchronous file I/O (use the async APIs).

## How It Actually Works

A Dart isolate is an independent event loop with its own heap and garbage collector. The Flutter engine runs your `main()`
on the *root isolate*, which is driven by the UI task runner: each vsync signal schedules a frame callback on that event
loop. Because one event loop runs one task at a time, a 245 ms synchronous function means the frame callback (and every
timer, input event and microtask) waits 245 ms.

`Isolate.run` spawns a new isolate in the same *isolate group* (sharing compiled code, which makes spawning cheap compared
to an independent isolate), runs the function there on a separate OS thread, sends the result back through a port, and
shuts the isolate down. Each isolate has its own heap, so garbage collection in the worker doesn't pause the UI isolate.
Messages are serialized by the VM into the receiving isolate's heap, except for objects it can safely share — which is why
"copying cost" is something to measure, not assume.

## Common mistakes

- **Thinking `async` means "background".** `Future(() => heavyWork())` still runs on the UI isolate.
- **Offloading tiny tasks** — spawn cost dominates.
- **Sending huge data both ways** when only a small result is needed.
- **Capturing `this` in the closure**, which tries to send the whole widget/state object (and fails on unsendable fields).
- **Calling plugins from a background isolate** without setting up the messenger.
- **Measuring in debug mode.** Debug builds are JIT-compiled with extra checks; profile/release are AOT-compiled and much
  faster. Decide based on profile mode.

## Exercise

1. Change `bigJson(200000)` to `20000` and `2000`. At what size does the main-isolate gap drop below one frame on your
   machine? Is offloading still worth it there?
2. Make `parseAndSum` return the full decoded list instead of a sum. How does `Isolate.run`'s time change, and why?
3. Build a long-lived worker with `Isolate.spawn` that receives search queries over a `SendPort` and replies with results, so
   the list is copied into the worker only once.
4. In a Flutter app, show a `CircularProgressIndicator` and parse the payload on button press, first directly and then with
   `compute`. Watch the spinner — then confirm with DevTools in profile mode.
