---
description: "Riverpod 3 for Flutter state: Provider, Notifier and AsyncNotifier, derived providers, AsyncValue loading/error states, and overrides in tests."
---

# 05 · Riverpod

Riverpod was written by the author of `provider` to fix the things that bite in larger apps: providers that
can't be found because they're in the wrong part of the widget tree (`ProviderNotFoundException`), awkward
dependencies between providers, and async loading states every widget re-implements. Its central idea is that
**providers are global declarations, but their state lives in a container**, not in the widget tree. This
lesson was run with flutter_riverpod 3.4.3. Riverpod's 2.x → 3.x migration changed several APIs (and older
tutorials use `StateNotifierProvider` and `StateProvider`, now considered legacy), so match examples to your
installed major version.

```bash
flutter pub add flutter_riverpod
```

## The provider types you'll actually use

| Declaration | State | Typical use |
|---|---|---|
| `Provider<T>((ref) => ...)` | a computed value | dependencies (a repository, an HTTP client), derived data |
| `NotifierProvider<N, T>(N.new)` | synchronous `T`, changed by methods on `N` | UI state: filters, selection, form state |
| `AsyncNotifierProvider<N, T>(N.new)` | `AsyncValue<T>` loaded in `build()`, then changed by methods | data from a backend that the user edits |
| `FutureProvider<T>` / `StreamProvider<T>` | `AsyncValue<T>` | read-only async data |

Add `.autoDispose` to any of them to destroy the state when nothing is listening (a details page's data,
for example), and `.family` to parameterize a provider by an argument (`userProvider(42)`).

## A todo list with loading, filtering and derived state

```dart title="riverpod_demo.dart"
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

// ---------- data layer ----------
class Todo {
  const Todo(this.id, this.title, {this.done = false});
  final int id;
  final String title;
  final bool done;
  Todo toggled() => Todo(id, title, done: !done);
}

abstract class TodoRepository {
  Future<List<Todo>> fetch();
}

class SlowFakeRepository implements TodoRepository {
  @override
  Future<List<Todo>> fetch() async {
    await Future<void>.delayed(const Duration(milliseconds: 300));
    return const [Todo(1, 'Write lesson'), Todo(2, 'Run tests', done: true)];
  }
}

// ---------- providers ----------
/// A plain Provider: a dependency other providers can read. Tests override it.
final repositoryProvider = Provider<TodoRepository>((ref) => SlowFakeRepository());

/// AsyncNotifier: loads asynchronously, then exposes methods that change the state.
class TodoList extends AsyncNotifier<List<Todo>> {
  @override
  Future<List<Todo>> build() => ref.watch(repositoryProvider).fetch();

  void toggle(int id) {
    final current = state.value;
    if (current == null) return;
    state = AsyncData([for (final t in current) t.id == id ? t.toggled() : t]);
  }

  void add(String title) {
    final current = state.value ?? const [];
    final nextId = current.fold(0, (m, t) => t.id > m ? t.id : m) + 1;
    state = AsyncData([...current, Todo(nextId, title)]);
  }
}

final todoListProvider = AsyncNotifierProvider<TodoList, List<Todo>>(TodoList.new);

enum Filter { all, open, done }

/// Synchronous state with a Notifier.
class FilterNotifier extends Notifier<Filter> {
  @override
  Filter build() => Filter.all;
  void set(Filter f) => state = f;
}

final filterProvider = NotifierProvider<FilterNotifier, Filter>(FilterNotifier.new);

/// Derived state: recomputes when either input changes.
final visibleTodosProvider = Provider<AsyncValue<List<Todo>>>((ref) {
  final filter = ref.watch(filterProvider);
  return ref.watch(todoListProvider).whenData((todos) => switch (filter) {
        Filter.all => todos,
        Filter.open => todos.where((t) => !t.done).toList(),
        Filter.done => todos.where((t) => t.done).toList(),
      });
});

final openCountProvider = Provider<int>(
  (ref) => ref.watch(todoListProvider).value?.where((t) => !t.done).length ?? 0,
);

// ---------- UI ----------
class TodoPage extends ConsumerWidget {
  const TodoPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final todos = ref.watch(visibleTodosProvider);
    final open = ref.watch(openCountProvider);
    return Scaffold(
      appBar: AppBar(title: Text('Todos ($open open)')),
      body: switch (todos) {
        AsyncData(:final value) => ListView(children: [
            for (final t in value)
              CheckboxListTile(
                title: Text(t.title),
                value: t.done,
                onChanged: (_) => ref.read(todoListProvider.notifier).toggle(t.id),
              ),
          ]),
        AsyncError(:final error) => Center(child: Text('Failed: $error')),
        _ => const Center(child: CircularProgressIndicator()),
      },
      bottomNavigationBar: SegmentedButton<Filter>(
        segments: const [
          ButtonSegment(value: Filter.all, label: Text('All')),
          ButtonSegment(value: Filter.open, label: Text('Open')),
          ButtonSegment(value: Filter.done, label: Text('Done')),
        ],
        selected: {ref.watch(filterProvider)},
        onSelectionChanged: (s) => ref.read(filterProvider.notifier).set(s.single),
      ),
    );
  }
}
```

The structure to notice:

- **`repositoryProvider`** is the dependency seam. Nothing else constructs a `SlowFakeRepository`; tests swap it.
- **`TodoList.build()`** is where async loading happens. It `ref.watch`es the repository provider — if that
  provider ever changed, the list would rebuild automatically. While the `Future` is pending, the provider's state
  is `AsyncLoading`; then `AsyncData` or `AsyncError`.
- **Methods replace `state`** with a new immutable value; Riverpod notifies listeners when you assign it.
  There's no `notifyListeners()` to forget.
- **`visibleTodosProvider` and `openCountProvider` are derived.** They watch other providers and recompute when
  those change — no manual syncing between "the list" and "the filtered list".
- **`ConsumerWidget`** gets a `WidgetRef`: `ref.watch` in `build` to subscribe, `ref.read` in callbacks to call
  methods — the same split as Provider's `watch`/`read`.
- The `switch` over `AsyncData` / `AsyncError` / loading uses Dart 3 pattern matching on `AsyncValue`'s sealed
  class hierarchy, so the compiler checks you handled the cases.

The whole app is wrapped once: `runApp(const ProviderScope(child: MyApp()))`. That `ProviderScope` owns the
container holding every provider's state.

## Tests: with and without widgets

```dart title="riverpod_test.dart"
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:l2/s/riverpod_demo.dart';

class InstantRepo implements TodoRepository {
  InstantRepo(this.items);
  final List<Todo> items;
  @override
  Future<List<Todo>> fetch() async => items;
}

class FailingRepo implements TodoRepository {
  @override
  Future<List<Todo>> fetch() async => throw StateError('offline');
}

void main() {
  test('providers without widgets: ProviderContainer', () async {
    final container = ProviderContainer.test(overrides: [
      repositoryProvider.overrideWithValue(InstantRepo(const [Todo(1, 'a'), Todo(2, 'b')])),
    ]);
    print('before load: ${container.read(todoListProvider)}');
    await container.read(todoListProvider.future);
    print('open after load: ${container.read(openCountProvider)}');
    container.read(todoListProvider.notifier)
      ..toggle(1)
      ..add('c');
    container.read(filterProvider.notifier).set(Filter.open);
    print('visible open: ${container.read(visibleTodosProvider).value!.map((t) => t.title).toList()}');
    print('open count: ${container.read(openCountProvider)}');
  });

  testWidgets('real repository: spinner, then data', (tester) async {
    await tester.pumpWidget(const ProviderScope(child: MaterialApp(home: TodoPage())));
    print('spinner: ${find.byType(CircularProgressIndicator).evaluate().length}');
    await tester.pump(const Duration(milliseconds: 300));
    print('rows: ${find.byType(CheckboxListTile).evaluate().length}, title: ${tester.widget<Text>(find.textContaining('open)')).data}');

    await tester.tap(find.text('Write lesson'));
    await tester.pump();
    print('after toggle: ${tester.widget<Text>(find.textContaining('open)')).data}');

    await tester.tap(find.text('Done'));
    await tester.pumpAndSettle();
    print('done filter rows: ${find.byType(CheckboxListTile).evaluate().length}');
  });

  testWidgets('error state via override', (tester) async {
    await tester.pumpWidget(ProviderScope(
      overrides: [repositoryProvider.overrideWithValue(FailingRepo())],
      retry: (retryCount, error) => null, // don't retry in this test
      child: const MaterialApp(home: TodoPage()),
    ));
    await tester.pump();
    print(tester.widget<Text>(find.textContaining('Failed')).data);
  });
}
```

```console
$ flutter test test/s/riverpod_test.dart
before load: AsyncLoading<List<Todo>>()
open after load: 2
visible open: [b, c]
open count: 2
spinner: 1
rows: 2, title: Todos (1 open)
after toggle: Todos (0 open)
done filter rows: 2
Failed: Bad state: offline
00:00 +3: All tests passed!
```

1. **No widgets needed.** `ProviderContainer.test()` creates a container (disposed automatically at the end of the
   test). Overriding `repositoryProvider` made loading instant; then toggling, adding and filtering were all
   checked by reading providers directly. Toggling `a` to done and adding `c` left `[b, c]` open.
2. **The real (slow) repository** showed a spinner on the first frame; after pumping 300 ms of fake time, two rows.
   Toggling "Write lesson" updated the AppBar count — `openCountProvider` recomputed because `todoListProvider`
   changed. The "Done" filter then showed both rows (both were done by then).
3. **The error path** was forced with an override. Riverpod 3 automatically **retries** providers that fail (with
   backoff) by default, which is great for flaky networks but would make this test wait; passing a `retry`
   function that returns `null` disables it for this scope.

## Riverpod versus Provider

Both are fine. Riverpod's advantages show up as an app grows: compile-time-safe access to any provider from
anywhere (no `ProviderNotFoundException`), dependencies between providers expressed with `ref.watch`, built-in
`AsyncValue`, automatic disposal, and testing by override. The costs: more concepts up front, and API changes
between majors. There's also an optional code-generation style (`@riverpod` annotations with
`riverpod_generator`) that writes the provider declarations for you; it's a matter of taste — this course uses the
manual declarations so you can see exactly what exists.

## How It Actually Works

A provider object like `todoListProvider` is an immutable *recipe*: it holds the function that creates the state,
not the state itself. The state lives in a `ProviderContainer` (`ProviderScope` creates one and exposes it through
an `InheritedWidget` — so Riverpod still uses the mechanism from lesson 03, just once at the root).

The first time something reads a provider, the container creates an **element** for it and runs `build`. Every
`ref.watch` made during `build` is recorded as an edge in a dependency graph. When a provider's state changes, the
container walks its dependents: derived providers are marked dirty and recomputed lazily the next time they're
read, and `ConsumerWidget`s that watched them are scheduled to rebuild. If a recomputed value is `==` to the
previous one, the propagation stops there. `autoDispose` providers are destroyed when their listener count drops
to zero; `overrides` simply tell the container to use a different recipe for a given provider.

## Common mistakes

- **`ref.watch` inside callbacks** (`onPressed`). Use `ref.read` there; `watch` belongs in `build`.
- **`ref.read` in `build` to display state** — the widget won't update.
- **Mutating state in place** (`state.value!.add(todo)`) — the reference didn't change, so nothing is notified.
  Always assign a new value.
- **Forgetting `ProviderScope`** at the root: the app throws on the first `ref.watch`.
- **Storing everything in one giant notifier.** Split by responsibility and derive the rest.
- **Copying 2.x examples into a 3.x app** (or vice versa). Check the major version first.

## Exercise

1. Add a text field and button that call `add`. Write a container test proving new todos get unique ids.
2. Make a `todoProvider = Provider.family<Todo?, int>` that returns one todo by id, and a details page that watches
   it. Use `.autoDispose` and verify (with `ref.onDispose` and a print) when it's destroyed.
3. Make `toggle` optimistic: update the UI immediately, call a (fake) `repository.save`, and roll back with an error
   message if it throws.
4. Port lesson 04's cart to Riverpod. Which Provider-era concepts map to which Riverpod ones?
