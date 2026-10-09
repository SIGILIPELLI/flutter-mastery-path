---
description: "The Bloc pattern in Flutter: Cubit vs Bloc, sealed events and states, a debounced search transformer, BlocBuilder/BlocListener and bloc_test."
---

# 07 · The Bloc Pattern

Bloc ("Business Logic Component") is the third major state-management approach you'll meet in Flutter codebases, common in
larger teams that like strict structure. Its rule is simple: **UI sends inputs in, the bloc emits states out**, and the only
way state changes is through those inputs. Every state transition is observable and testable as a sequence. This lesson used
flutter_bloc 9.1.1 (bloc 9.2.1) and bloc_test 10.0.0.

```bash
flutter pub add flutter_bloc
flutter pub add --dev bloc_test
```

## Cubit and Bloc

| | Input | Good for |
|---|---|---|
| `Cubit<State>` | method calls (`cubit.toggle()`) that `emit` new states | simple state: a theme, a counter, a form toggle |
| `Bloc<Event, State>` | event objects (`bloc.add(QueryChanged('he'))`) handled by `on<Event>` | flows where *how* events are processed matters: debouncing, cancelling, ordering, logging every input |

A Cubit is a Bloc without the event layer. Start with Cubit; upgrade to Bloc when you need event transformations or a log of
inputs.

## A theme Cubit and a search Bloc

```dart title="bloc_demo.dart"
import 'dart:async';
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

// ---------- Cubit: methods in, states out ----------
class ThemeCubit extends Cubit<ThemeMode> {
  ThemeCubit() : super(ThemeMode.system);
  void toggle() => emit(state == ThemeMode.dark ? ThemeMode.light : ThemeMode.dark);
}

// ---------- Bloc: events in, states out ----------
sealed class SearchEvent {}

final class QueryChanged extends SearchEvent {
  QueryChanged(this.query);
  final String query;
}

sealed class SearchState {
  const SearchState();
}

final class SearchIdle extends SearchState {
  const SearchIdle();
  @override
  String toString() => 'Idle';
}

final class SearchLoading extends SearchState {
  const SearchLoading(this.query);
  final String query;
  @override
  String toString() => 'Loading($query)';
}

final class SearchResults extends SearchState {
  const SearchResults(this.query, this.items);
  final String query;
  final List<String> items;
  @override
  String toString() => 'Results($query: $items)';
}

final class SearchFailure extends SearchState {
  const SearchFailure(this.message);
  final String message;
  @override
  String toString() => 'Failure($message)';
}

abstract class CityApi {
  Future<List<String>> search(String q);
}

/// Waits for [duration] of quiet before passing the latest event on.
EventTransformer<E> debounce<E>(Duration duration) => (events, mapper) {
      final controller = StreamController<E>();
      Timer? timer;
      final sub = events.listen(
        (e) {
          timer?.cancel();
          timer = Timer(duration, () => controller.add(e));
        },
        onDone: () async {
          timer?.cancel();
          await controller.close();
        },
      );
      controller.onCancel = () {
        timer?.cancel();
        return sub.cancel();
      };
      return controller.stream.asyncExpand(mapper); // one search at a time, in order
    };

class SearchBloc extends Bloc<SearchEvent, SearchState> {
  SearchBloc(this._api) : super(const SearchIdle()) {
    on<QueryChanged>(_onQuery, transformer: debounce(const Duration(milliseconds: 300)));
  }
  final CityApi _api;

  Future<void> _onQuery(QueryChanged e, Emitter<SearchState> emit) async {
    final q = e.query.trim();
    if (q.isEmpty) return emit(const SearchIdle());
    emit(SearchLoading(q));
    try {
      emit(SearchResults(q, await _api.search(q)));
    } catch (err) {
      emit(SearchFailure('$err'));
    }
  }
}

// ---------- UI ----------
class SearchPage extends StatelessWidget {
  const SearchPage({super.key});
  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: const Text('Cities')),
        body: Column(children: [
          TextField(key: const Key('q'), onChanged: (q) => context.read<SearchBloc>().add(QueryChanged(q))),
          // BlocListener: one-off side effects (snackbars, navigation), never builds UI.
          BlocListener<SearchBloc, SearchState>(
            listenWhen: (prev, next) => next is SearchFailure,
            listener: (context, state) => ScaffoldMessenger.of(context)
                .showSnackBar(SnackBar(content: Text((state as SearchFailure).message))),
            child: const SizedBox.shrink(),
          ),
          Expanded(
            child: BlocBuilder<SearchBloc, SearchState>(
              builder: (context, state) => switch (state) {
                SearchIdle() => const Center(child: Text('Type to search')),
                SearchLoading() => const Center(child: CircularProgressIndicator()),
                SearchResults(:final items) when items.isEmpty => const Center(child: Text('No matches')),
                SearchResults(:final items) => ListView(children: [for (final c in items) ListTile(title: Text(c))]),
                SearchFailure() => const Center(child: Text('Something went wrong')),
              },
            ),
          ),
        ]),
      );
}
```

Things to notice:

- **Sealed classes for events and states.** The `switch` in `BlocBuilder` is checked by the compiler: add a new state
  subclass and every switch that doesn't handle it becomes an error. That's Bloc's structure paying off with Dart 3.
- **The handler (`_onQuery`) is ordinary async code** that calls `emit` as it goes: `Loading`, then `Results` or `Failure`.
- **The transformer decides how events flow into the handler.** Ours waits for 300 ms of quiet and then processes events one
  at a time (`asyncExpand`). Without a transformer, Bloc processes events *concurrently* by default. The
  `bloc_concurrency` package provides ready-made `sequential()`, `droppable()` and `restartable()` transformers — restartable
  (cancel the previous search when a new query arrives) is often the best fit for search.
- **`BlocBuilder` builds UI from state; `BlocListener` performs side effects** (snackbar, navigation, analytics) once per
  matching state change. Don't show snackbars from a builder — builders can run many times for one state.
- `context.read<SearchBloc>()` in callbacks, provided by `BlocProvider` (Bloc builds on `provider` under the hood).

## Testing state sequences

`blocTest` builds a bloc, acts on it, waits, and asserts on the exact list of emitted states:

```dart title="bloc_test.dart"
import 'package:bloc_test/bloc_test.dart';
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:l2/a/bloc_demo.dart';

class FakeCityApi implements CityApi {
  final queries = <String>[];
  @override
  Future<List<String>> search(String q) async {
    queries.add(q);
    if (q == 'boom') throw Exception('server down');
    const all = ['Hyderabad', 'Helsinki', 'Hanoi', 'Lima', 'Lisbon'];
    return all.where((c) => c.toLowerCase().startsWith(q.toLowerCase())).toList();
  }
}

void main() {
  blocTest<ThemeCubit, ThemeMode>(
    'cubit toggles',
    build: ThemeCubit.new,
    act: (c) => c..toggle()..toggle(),
    expect: () => [ThemeMode.dark, ThemeMode.light],
  );

  late FakeCityApi api;
  blocTest<SearchBloc, SearchState>(
    'typing fast searches once, with the final query',
    setUp: () => api = FakeCityApi(),
    build: () => SearchBloc(api),
    act: (b) async {
      for (final q in ['h', 'he', 'hel']) {
        b.add(QueryChanged(q));
        await Future<void>.delayed(const Duration(milliseconds: 50));
      }
    },
    wait: const Duration(milliseconds: 400),
    expect: () => [isA<SearchLoading>(), isA<SearchResults>()],
    verify: (b) => print('api calls: ${api.queries}; final: ${b.state}'),
  );

  blocTest<SearchBloc, SearchState>(
    'failure and recovery',
    setUp: () => api = FakeCityApi(),
    build: () => SearchBloc(api),
    act: (b) async {
      b.add(QueryChanged('boom'));
      await Future<void>.delayed(const Duration(milliseconds: 400));
      b.add(QueryChanged('li'));
    },
    wait: const Duration(milliseconds: 400),
    verify: (b) => print('final: ${b.state}'),
    expect: () => [isA<SearchLoading>(), isA<SearchFailure>(), isA<SearchLoading>(), isA<SearchResults>()],
  );

  testWidgets('widget: builder renders states, listener shows a snackbar', (tester) async {
    final bloc = SearchBloc(FakeCityApi());
    final seen = <String>[];
    final sub = bloc.stream.listen((s) => seen.add('$s'));
    addTearDown(() async {
      await sub.cancel();
      await bloc.close();
    });
    await tester.pumpWidget(MaterialApp(home: BlocProvider.value(value: bloc, child: const SearchPage())));
    await tester.enterText(find.byKey(const Key('q')), 'ha');
    await tester.pump(const Duration(milliseconds: 300)); // debounce fires
    await tester.pump(); // results arrive
    print('rows: ${find.byType(ListTile).evaluate().map((e) => ((e.widget as ListTile).title as Text).data).toList()}');
    await tester.enterText(find.byKey(const Key('q')), 'boom');
    await tester.pump(const Duration(milliseconds: 300));
    await tester.pump();
    await tester.pump(const Duration(milliseconds: 750)); // snackbar entrance animation
    print('snackbar: ${find.text('Exception: server down').evaluate().length}, body: ${find.text('Something went wrong').evaluate().length}');
    print('states: $seen');
  });
}
```

```console
$ flutter test test/a/bloc_test.dart
api calls: [hel]; final: Results(hel: [Helsinki])
final: Results(li: [Lima, Lisbon])
rows: [Hanoi]
snackbar: 1, body: 1
states: [Loading(ha), Results(ha: [Hanoi]), Loading(boom), Failure(Exception: server down)]
00:02 +4: All tests passed!
```

- **Cubit:** two toggles from `system` produced exactly `[dark, light]`.
- **Debounce:** three keystrokes 50 ms apart produced **one** API call, with the final query `hel`, and exactly two states.
  Without the transformer, Bloc's default concurrent handling would call the API for every keystroke, and a slow early
  response could arrive after a later one.
- **Failure and recovery:** the failing query emitted `Loading → Failure`; the next query recovered to `Results`. The bloc
  isn't "stuck" after an error because errors are modelled as an ordinary state.
- **Widget test:** the full UI path — typing, debounce, results rendered; then a failure rendered in the body *and* reported
  once through the `BlocListener` snackbar.

`blocTest` runs on real time (that's why the file took about two seconds). Keep debounce durations short in tests or inject
them.

## Bloc, Riverpod, Provider — choosing

All three are maintained and used in production. Rough guidance:

- **Provider + ChangeNotifier**: least ceremony; fine for small/medium apps ([Level 2 · 04](../level-2/04-provider.md)).
- **Riverpod**: dependency graph, async state and overrides built in; flexible ([Level 2 · 05](../level-2/05-riverpod.md)).
- **Bloc**: explicit events and states, strong conventions, excellent for teams that want every transition traceable
  (`BlocObserver` can log every event and state in the app).

Consistency within one app matters more than the choice. Mixing all three is the real anti-pattern.

## How It Actually Works

A `Bloc` owns a `StreamController` of events. `on<E>(handler, transformer: t)` registers a handler: the bloc filters the event
stream to type `E`, applies the transformer `t` — a function from `(Stream<E> events, mapper)` to `Stream` — and the mapper
turns each event into a stream that runs your handler with an `Emitter`. That's how one function controls concurrency:
`asyncExpand` processes the inner streams one after another, a "switchMap"-style transformer cancels the previous inner stream,
and the default merges them concurrently.

`emit` checks the new state against the current one and skips it if they're equal (`==`), then pushes it onto the bloc's
state stream and updates `state`. If the handler has completed or been cancelled, its `Emitter` is closed and late `emit`
calls are ignored (with an assertion in debug mode if you emit after the handler finished without awaiting). `BlocBuilder`
subscribes to the state stream and calls `setState`; `buildWhen`/`listenWhen` filter transitions before rebuilding.

## Common mistakes

- **Mutable states.** Emitting the same (mutated) object is skipped by the equality check, so the UI doesn't update. Create new
  state objects; consider `equatable` or records for value equality.
- **Side effects in `BlocBuilder`.** Use `BlocListener` (or `BlocConsumer`, which combines both).
- **One giant bloc per app.** Scope blocs to features and provide them where they're needed.
- **Forgetting `await` before calling `emit`** in a handler, so it runs after the handler completed.
- **Not closing blocs you created manually** (`BlocProvider(create: ...)` closes them for you; `.value` does not).
- **Leaving out a transformer for search-like events** and getting out-of-order results.

## Exercise

1. Replace the hand-written `debounce` with `bloc_concurrency`'s `restartable()` combined with a debounce. Write a test where a
   slow search for `he` is overtaken by a fast search for `hel` — only `hel`'s results may be emitted.
2. Add a `SearchCleared` event that resets to `Idle` immediately, bypassing the debounce.
3. Write a `BlocObserver` that prints every transition, install it in `main`, and use it to trace one user session.
4. Port the Level 2 reading-list notifier to a Bloc. Which events did you need? Was the result clearer or just longer?
