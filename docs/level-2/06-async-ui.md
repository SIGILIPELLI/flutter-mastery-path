---
description: "Render asynchronous data with FutureBuilder and StreamBuilder: snapshot states, retry, the future-in-build trap, and fake-time widget tests."
---

# 06 · Async UI: FutureBuilder & StreamBuilder

Most screens wait for something: a network response, a database query, a stream of location updates. Dart
models these as `Future<T>` (one value, later) and `Stream<T>` (many values over time) — if those are new, the
[Dart course](https://sigilipelli.github.io/dart-mastery-path/) covers them. Flutter's two built-in widgets for
turning them into UI are `FutureBuilder` and `StreamBuilder`. They're simple, and they have one trap that catches
nearly everyone; this lesson walks into it on purpose.

## Snapshots

Both builders call your `builder` with an `AsyncSnapshot<T>` every time something changes. Its fields:

| Field | Meaning |
|---|---|
| `connectionState` | `none` (no future/stream), `waiting` (nothing yet), `active` (stream has emitted, still open), `done` |
| `hasData` / `data` | the latest value, if any |
| `hasError` / `error` / `stackTrace` | the error, if the future failed or the stream emitted one |

A future goes `waiting → done`. A stream goes `waiting → active → active … → done`, and `data` holds the *latest*
event, even after `done`.

## The demo

```dart title="async_demo.dart"
import 'dart:async';
import 'package:flutter/material.dart';

Future<String> fetchQuote({bool fail = false}) async {
  await Future<void>.delayed(const Duration(seconds: 1));
  if (fail) throw TimeoutException('quote server did not answer');
  return 'Simplicity is prerequisite for reliability.';
}

class QuoteCard extends StatefulWidget {
  const QuoteCard({super.key, this.fail = false});
  final bool fail;
  @override
  State<QuoteCard> createState() => _QuoteCardState();
}

class _QuoteCardState extends State<QuoteCard> {
  // Created once, in initState — NOT in build.
  late Future<String> _quote = fetchQuote(fail: widget.fail);

  void _retry() {
    // Block body on purpose: `setState(() => _quote = ...)` would return the Future
    // from the closure, and setState asserts against that.
    setState(() {
      _quote = fetchQuote();
    });
  }

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<String>(
      future: _quote,
      builder: (context, snapshot) {
        if (snapshot.connectionState != ConnectionState.done) {
          return const CircularProgressIndicator();
        }
        if (snapshot.hasError) {
          return Column(children: [
            Text('Error: ${snapshot.error}'),
            TextButton(onPressed: _retry, child: const Text('Retry')),
          ]);
        }
        return Text(snapshot.data!);
      },
    );
  }
}

/// A countdown as a Stream.
Stream<int> countdown(int from) async* {
  for (var i = from; i >= 0; i--) {
    yield i;
    if (i > 0) await Future<void>.delayed(const Duration(seconds: 1));
  }
}

class CountdownView extends StatefulWidget {
  const CountdownView({super.key});
  @override
  State<CountdownView> createState() => _CountdownViewState();
}

class _CountdownViewState extends State<CountdownView> {
  final _ticks = countdown(3);
  @override
  Widget build(BuildContext context) => StreamBuilder<int>(
        stream: _ticks,
        builder: (context, snap) => Text(switch (snap) {
          AsyncSnapshot(connectionState: ConnectionState.waiting) => 'waiting',
          AsyncSnapshot(connectionState: ConnectionState.done, :final data) => 'liftoff (last value $data)',
          AsyncSnapshot(:final data?) => 'T-$data',
          _ => '?',
        }, key: const Key('cd')),
      );
}

/// The bug version: a new Future on every build.
class LeakyQuote extends StatelessWidget {
  const LeakyQuote({super.key, required this.onFetch});
  final VoidCallback onFetch;
  @override
  Widget build(BuildContext context) => FutureBuilder<String>(
        future: () {
          onFetch();
          return fetchQuote();
        }(),
        builder: (context, s) => Text(s.data ?? 'loading'),
      );
}
```

Things to notice in `QuoteCard`:

- The future is created **once**, by a `late` field initializer on the `State`. A `late` field is initialized on
  first access — here, the first `build` — and then kept for the life of the `State`, so later rebuilds reuse the
  same future. (Assigning it in `initState` is equivalent and more explicit.) It isn't created inside `build`.
- `connectionState != done` covers both "waiting" and the moment between frames; error is checked before data.
- Retry assigns a **new** future inside `setState`. `FutureBuilder` notices the different future and starts over
  from `waiting`.

`CountdownView` uses `async*` to produce a stream and a Dart 3 `switch` on the snapshot's fields to pick the label.

## Running it with fake time

```dart title="async_test.dart"
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:l2/s/async_demo.dart';

void main() {
  testWidgets('FutureBuilder: loading, then data', (tester) async {
    await tester.pumpWidget(const MaterialApp(home: QuoteCard()));
    print('t=0s   spinner=${find.byType(CircularProgressIndicator).evaluate().length}');
    await tester.pump(const Duration(seconds: 1));
    print('t=1s   ${tester.widget<Text>(find.byType(Text)).data}');
  });

  testWidgets('FutureBuilder: error and retry', (tester) async {
    await tester.pumpWidget(const MaterialApp(home: Scaffold(body: QuoteCard(fail: true))));
    await tester.pump(const Duration(seconds: 1));
    print('error  ${tester.widget<Text>(find.textContaining('Error')).data}');
    await tester.tap(find.text('Retry'));
    await tester.pump();
    print('retry  spinner=${find.byType(CircularProgressIndicator).evaluate().length}');
    await tester.pump(const Duration(seconds: 1));
    print('then   ${find.textContaining('Simplicity').evaluate().length} quote shown');
  });

  testWidgets('StreamBuilder countdown', (tester) async {
    await tester.pumpWidget(const MaterialApp(home: CountdownView()));
    String label() => tester.widget<Text>(find.byKey(const Key('cd'))).data!;
    final seen = <String>[label()];
    await tester.pump(); // deliver the first event (3) in its own frame
    seen.add(label());
    for (var i = 0; i < 4; i++) {
      await tester.pump(const Duration(seconds: 1));
      seen.add(label());
    }
    print('countdown frames: $seen');
  });

  testWidgets('creating the future in build refetches on every rebuild', (tester) async {
    var fetches = 0;
    Widget app(int n) => MaterialApp(home: Column(children: [Text('rebuild $n'), LeakyQuote(onFetch: () => fetches++)]));
    for (var n = 0; n < 3; n++) {
      await tester.pumpWidget(app(n));
    }
    await tester.pump(const Duration(seconds: 1));
    print('3 rebuilds -> $fetches fetches');
  });
}
```

```console
$ flutter test test/s/async_test.dart
t=0s   spinner=1
t=1s   Simplicity is prerequisite for reliability.
error  Error: TimeoutException: quote server did not answer
retry  spinner=1
then   1 quote shown
countdown frames: [waiting, T-3, T-2, T-1, liftoff (last value 0), liftoff (last value 0)]
3 rebuilds -> 3 fetches
00:00 +4: All tests passed!
```

Widget tests run on a **fake clock**: `tester.pump(const Duration(seconds: 1))` advances time by exactly one second
and runs one frame, so a one-second delay finishes instantly and deterministically. The whole file ran in under a
second of real time.

- The quote card showed a spinner, then the quote after one (fake) second.
- The failing version showed the error; Retry put the spinner back, and the next second produced the quote.
- The countdown stream's first frame was `waiting`; each later frame showed the latest event. `0` never appears
  as `T-0`: the generator yields `0` and immediately finishes, so by the next frame the snapshot is already `done`
  (with `data` still `0`). Streams can deliver several events between two frames — the builder only ever sees the
  latest.
- The last test is the trap.

## The trap: creating the future in `build`

`LeakyQuote` creates its future *inside* `build`. Every rebuild — a parent's `setState`, a theme change, the keyboard
opening — makes a **new** future, and `FutureBuilder` dutifully restarts: it shows the loading state again and the
request is sent again. Three rebuilds, three fetches. In a real app that's a flickering spinner and a hammered API.

The fix is always the same: create the future somewhere that survives rebuilds (a `State` field, as in
`QuoteCard`, or a state-management layer like [Riverpod's `AsyncNotifier`](05-riverpod.md), which also caches it), and
hand that same object to the builder.

## A bug we hit while writing this lesson

The first version of `_retry` was

```dart
void _retry() => setState(() => _quote = fetchQuote());
```

and tapping Retry threw:

```console
setState() callback argument returned a Future.
The setState() method on _QuoteCardState#1b711 was called with a closure or method that returned a
Future. Maybe it is marked as "async".
```

An arrow function returns its expression's value, and an assignment's value is the assigned value — here, a
`Future`. `setState` asserts that its callback doesn't return a `Future`, because that usually means someone wrote
`setState(() async { ... })` and expected the UI to wait. Using a block body (`setState(() { _quote = ...; })`)
returns nothing and fixes it.

## When to reach for something else

`FutureBuilder`/`StreamBuilder` are ideal for local, self-contained async UI. Once several widgets need the same
data, or you need caching, refresh, pagination or optimistic updates, move the async work into your state layer
(Provider, Riverpod, Bloc) and let widgets render its state. The snapshot-handling discipline stays the same.

## How It Actually Works

`FutureBuilder` is a `StatefulWidget`. When its state is initialized (or `didUpdateWidget` sees a **different**
future object than before, compared by identity), it calls `future.then(...)` and stores a snapshot with
`connectionState: waiting`. When the future completes, the callback calls `setState` with a `done` snapshot — but
only if that future is still the current one, so a slow old request can't overwrite a newer result. `StreamBuilder`
does the same with `stream.listen`, updating the snapshot on each event, and **cancels its subscription** when the
stream changes or the widget is disposed.

In tests, `flutter_test` replaces the real timer zone with a `FakeAsync` clock. `Future.delayed` registers a fake
timer; `pump(duration)` advances the clock, fires due timers, flushes microtasks and renders a frame. A test that
ends with timers still pending fails with "A Timer is still pending even after the widget tree was disposed" — a
useful signal that something kept running.

## Common mistakes

- **Creating the future/stream in `build`.** Shown above.
- **Checking `hasData` first and showing data during a refresh** without signalling the refresh — decide
  deliberately whether stale data or a spinner is better.
- **Not handling `hasError`**, so a failed request leaves a spinner forever.
- **Single-subscription streams used twice**: listening again throws "Stream has already been listened to". Keep
  one stream in state, or use a broadcast stream.
- **`setState` after `dispose`** when calling async code by hand (outside a builder): check `mounted` after awaits.
- **`pumpAndSettle` with an infinite animation** (like an indeterminate `CircularProgressIndicator`) times out;
  pump explicit durations while a spinner is visible.

## Exercise

1. Add a "Refresh" button to the quote card that keeps showing the *old* quote, with a thin `LinearProgressIndicator`
   above it, while the new one loads. Hint: keep the last data in the `State`.
2. Write a test proving a slow first request that finishes *after* a retry's fast request can't overwrite it.
3. Turn `countdown` into a broadcast stream and show it in two `StreamBuilder`s. What changes about when it starts?
4. Make the countdown restartable from a button without leaking subscriptions.
