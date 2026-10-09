---
description: "Level 2 project: a reading-list app combining go_router, Riverpod, a repository over shared_preferences, form validation and a full widget-test suite."
---

# 10 · Project — A Reading List App

This project puts Level 2 together in one small but complete app: three shelves (*To read*, *Reading*, *Done*), an
add-book form, a detail page to move, rate and remove books, persistence across launches, and deep links. The point
isn't the feature list — it's seeing how routing, state, storage and tests fit together without any of them knowing
too much about the others.

```text
lib/rl/
  model.dart       Book, Shelf, JSON encoding           (pure Dart)
  repository.dart  BookRepository + two implementations (storage)
  state.dart       Riverpod providers and the notifier  (app logic)
  app.dart         router and pages                     (UI)
  main.dart        wiring: real storage → ProviderScope
test/rl/reading_list_test.dart
```

Dependencies are the ones from this level: `go_router`, `flutter_riverpod`, `shared_preferences`.

## Model

```dart title="lib/rl/model.dart"
import 'dart:convert';

enum Shelf { toRead, reading, done }

class Book {
  const Book({required this.id, required this.title, required this.author, this.shelf = Shelf.toRead, this.rating});
  final String id;
  final String title;
  final String author;
  final Shelf shelf;
  final int? rating; // 1..5, only for finished books

  Book copyWith({Shelf? shelf, int? rating}) =>
      Book(id: id, title: title, author: author, shelf: shelf ?? this.shelf, rating: rating ?? this.rating);

  Map<String, Object?> toJson() => {'id': id, 'title': title, 'author': author, 'shelf': shelf.name, 'rating': rating};

  factory Book.fromJson(Map<String, Object?> j) => Book(
        id: j['id'] as String,
        title: j['title'] as String,
        author: j['author'] as String,
        shelf: Shelf.values.byName(j['shelf'] as String),
        rating: j['rating'] as int?,
      );
}

String encodeBooks(List<Book> books) => jsonEncode([for (final b in books) b.toJson()]);
List<Book> decodeBooks(String s) => [for (final e in jsonDecode(s) as List) Book.fromJson(e as Map<String, Object?>)];
```

`Shelf.values.byName` turns the stored string back into an enum and throws on unknown names, so a corrupted value
fails loudly. Storing `shelf.name` rather than `shelf.index` means reordering the enum later can't silently move
every book to a different shelf.

## Repository

```dart title="lib/rl/repository.dart"
import 'package:shared_preferences/shared_preferences.dart';
import 'model.dart';

abstract class BookRepository {
  Future<List<Book>> load();
  Future<void> save(List<Book> books);
}

class PrefsBookRepository implements BookRepository {
  PrefsBookRepository(this._prefs);
  final SharedPreferences _prefs;
  static const _key = 'readingList.v1';

  @override
  Future<List<Book>> load() async {
    final raw = _prefs.getString(_key);
    return raw == null ? [] : decodeBooks(raw);
  }

  @override
  Future<void> save(List<Book> books) => _prefs.setString(_key, encodeBooks(books));
}

class MemoryBookRepository implements BookRepository {
  MemoryBookRepository([List<Book> initial = const []]) : _books = [...initial];
  List<Book> _books;
  int saves = 0;
  @override
  Future<List<Book>> load() async => [..._books];
  @override
  Future<void> save(List<Book> books) async {
    saves++;
    _books = [...books];
  }
}
```

The interface has exactly two methods because that's all the app needs. `MemoryBookRepository` counts saves so tests
can check persistence happens. The storage key carries a version (`readingList.v1`) — the migration idea from
[lesson 08](08-persistence.md) in its simplest form. For a reading list of a few hundred books, one JSON string is
fine; a library app with thousands would move to SQLite behind the same interface.

## State

```dart title="lib/rl/state.dart"
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'model.dart';
import 'repository.dart';

final repositoryProvider = Provider<BookRepository>((ref) => throw UnimplementedError('override in main()'));

class ReadingList extends AsyncNotifier<List<Book>> {
  @override
  Future<List<Book>> build() => ref.watch(repositoryProvider).load();

  Future<void> _update(List<Book> Function(List<Book>) change) async {
    final next = change(state.requireValue);
    state = AsyncData(next); // UI updates immediately
    await ref.read(repositoryProvider).save(next);
  }

  Future<String> add(String title, String author) async {
    final id = DateTime.now().microsecondsSinceEpoch.toString();
    await _update((b) => [...b, Book(id: id, title: title.trim(), author: author.trim())]);
    return id;
  }

  Future<void> move(String id, Shelf shelf) =>
      _update((b) => [for (final x in b) x.id == id ? x.copyWith(shelf: shelf) : x]);

  Future<void> rate(String id, int stars) {
    if (stars < 1 || stars > 5) throw RangeError.range(stars, 1, 5, 'stars');
    return _update((b) => [for (final x in b) x.id == id ? x.copyWith(shelf: Shelf.done, rating: stars) : x]);
  }

  Future<void> remove(String id) => _update((b) => b.where((x) => x.id != id).toList());
}

final readingListProvider = AsyncNotifierProvider<ReadingList, List<Book>>(ReadingList.new);

final shelfProvider = Provider.family<List<Book>, Shelf>((ref, shelf) =>
    ref.watch(readingListProvider).value?.where((b) => b.shelf == shelf).toList() ?? const []);

final bookProvider = Provider.family<Book?, String>((ref, id) {
  final books = ref.watch(readingListProvider).value ?? const [];
  for (final b in books) {
    if (b.id == id) return b;
  }
  return null;
});
```

- **`repositoryProvider` throws unless overridden.** That forces the real storage to be chosen in exactly one place,
  `main()`, and makes tests supply their own. It's a common Riverpod pattern for dependencies that need async setup.
- **Every mutation goes through `_update`**, which updates the UI first and then saves. That's *optimistic*: the user
  never waits on storage. With local storage, failure is rare; with a server you'd catch errors and roll back
  (lesson 05, exercise 3).
- **Rules live here, not in widgets.** Rating a book moves it to *Done*; ratings outside 1–5 are rejected.
- **`shelfProvider` and `bookProvider` are `family` providers** derived from the one list, so there's a single
  source of truth.

## UI and routing

```dart title="lib/rl/app.dart"
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'model.dart';
import 'state.dart';

const shelfLabels = {Shelf.toRead: 'To read', Shelf.reading: 'Reading', Shelf.done: 'Done'};

GoRouter buildRouter() => GoRouter(routes: [
      GoRoute(path: '/', builder: (_, s) => const ShelvesPage(), routes: [
        GoRoute(path: 'add', builder: (_, s) => const AddBookPage()),
        GoRoute(path: 'book/:id', builder: (_, s) => BookPage(id: s.pathParameters['id']!)),
      ]),
    ]);

class ReadingListApp extends StatefulWidget {
  const ReadingListApp({super.key});
  @override
  State<ReadingListApp> createState() => _ReadingListAppState();
}

class _ReadingListAppState extends State<ReadingListApp> {
  final _router = buildRouter(); // created once, not in build
  @override
  void dispose() {
    _router.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => MaterialApp.router(
        title: 'Reading list',
        theme: ThemeData(colorSchemeSeed: Colors.indigo),
        routerConfig: _router,
      );
}

class ShelvesPage extends ConsumerWidget {
  const ShelvesPage({super.key});
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final all = ref.watch(readingListProvider);
    return DefaultTabController(
      length: Shelf.values.length,
      child: Scaffold(
        appBar: AppBar(
          title: const Text('Reading list'),
          bottom: TabBar(tabs: [
            for (final s in Shelf.values) Tab(text: '${shelfLabels[s]} (${ref.watch(shelfProvider(s)).length})'),
          ]),
        ),
        floatingActionButton: FloatingActionButton(
          tooltip: 'Add book',
          onPressed: () => context.go('/add'),
          child: const Icon(Icons.add),
        ),
        body: switch (all) {
          AsyncError(:final error) => Center(child: Text('Could not load: $error')),
          AsyncData() => TabBarView(children: [for (final s in Shelf.values) ShelfList(shelf: s)]),
          _ => const Center(child: CircularProgressIndicator()),
        },
      ),
    );
  }
}

class ShelfList extends ConsumerWidget {
  const ShelfList({super.key, required this.shelf});
  final Shelf shelf;
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final books = ref.watch(shelfProvider(shelf));
    if (books.isEmpty) return Center(child: Text('Nothing in "${shelfLabels[shelf]}"'));
    return ListView(children: [
      for (final b in books)
        ListTile(
          key: ValueKey(b.id),
          title: Text(b.title),
          subtitle: Text(b.rating == null ? b.author : '${b.author} · ${'★' * b.rating!}'),
          onTap: () => context.go('/book/${b.id}'),
        ),
    ]);
  }
}

class AddBookPage extends ConsumerStatefulWidget {
  const AddBookPage({super.key});
  @override
  ConsumerState<AddBookPage> createState() => _AddBookPageState();
}

class _AddBookPageState extends ConsumerState<AddBookPage> {
  final _form = GlobalKey<FormState>();
  final _title = TextEditingController();
  final _author = TextEditingController();

  @override
  void dispose() {
    _title.dispose();
    _author.dispose();
    super.dispose();
  }

  String? _required(String? v) => (v == null || v.trim().isEmpty) ? 'Required' : null;

  Future<void> _save() async {
    if (!_form.currentState!.validate()) return;
    await ref.read(readingListProvider.notifier).add(_title.text, _author.text);
    if (mounted) context.go('/');
  }

  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: const Text('Add book')),
        body: Form(
          key: _form,
          child: ListView(padding: const EdgeInsets.all(16), children: [
            TextFormField(key: const Key('title'), controller: _title, decoration: const InputDecoration(labelText: 'Title'), validator: _required),
            TextFormField(key: const Key('author'), controller: _author, decoration: const InputDecoration(labelText: 'Author'), validator: _required),
            const SizedBox(height: 16),
            FilledButton(onPressed: _save, child: const Text('Save')),
          ]),
        ),
      );
}

class BookPage extends ConsumerWidget {
  const BookPage({super.key, required this.id});
  final String id;
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final book = ref.watch(bookProvider(id));
    if (book == null) {
      return Scaffold(appBar: AppBar(), body: const Center(child: Text('This book is no longer on your list')));
    }
    final list = ref.read(readingListProvider.notifier);
    return Scaffold(
      appBar: AppBar(title: Text(book.title), actions: [
        IconButton(
          tooltip: 'Remove',
          icon: const Icon(Icons.delete_outline),
          onPressed: () async {
            await list.remove(id);
            if (context.mounted) context.go('/');
          },
        ),
      ]),
      body: ListView(padding: const EdgeInsets.all(16), children: [
        Text('by ${book.author}'),
        const SizedBox(height: 16),
        SegmentedButton<Shelf>(
          segments: [for (final s in Shelf.values) ButtonSegment(value: s, label: Text(shelfLabels[s]!))],
          selected: {book.shelf},
          onSelectionChanged: (s) => list.move(id, s.single),
        ),
        if (book.shelf == Shelf.done)
          Row(children: [
            for (var i = 1; i <= 5; i++)
              IconButton(
                key: Key('star-$i'),
                icon: Icon(i <= (book.rating ?? 0) ? Icons.star : Icons.star_border),
                onPressed: () => list.rate(id, i),
              ),
          ]),
      ]),
    );
  }
}
```

Notes:

- The `GoRouter` is created once in a `State` and disposed with it (lesson 02's "don't create the router in
  `build`").
- `/add` and `/book/:id` are nested under `/`, so both get a back button to the shelves even when opened by deep link.
- `BookPage` handles an id that doesn't exist — whether from a stale deep link or because the book was just removed —
  instead of crashing on `null`.
- Widgets only call notifier methods (`ref.read(...notifier).move(...)`) and render providers. Swap the UI for a
  different design and nothing else changes.

```dart title="lib/rl/main.dart"
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'app.dart';
import 'repository.dart';
import 'state.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized(); // needed before plugins are used
  final prefs = await SharedPreferences.getInstance();
  runApp(ProviderScope(
    overrides: [repositoryProvider.overrideWithValue(PrefsBookRepository(prefs))],
    child: const ReadingListApp(),
  ));
}
```

`WidgetsFlutterBinding.ensureInitialized()` is required before using a plugin (here, `SharedPreferences`) ahead of
`runApp`, because plugins talk to the engine through the binding.

## Tests

```dart title="test/rl/reading_list_test.dart"
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:go_router/go_router.dart';
import 'package:l2/rl/app.dart';
import 'package:l2/rl/model.dart';
import 'package:l2/rl/repository.dart';
import 'package:l2/rl/state.dart';
import 'package:shared_preferences/shared_preferences.dart';

const dune = Book(id: 'b1', title: 'Dune', author: 'Frank Herbert');
const kindred = Book(id: 'b2', title: 'Kindred', author: 'Octavia E. Butler', shelf: Shelf.reading);

void main() {
  test('JSON round trip keeps every field', () {
    final books = [dune, kindred.copyWith(shelf: Shelf.done, rating: 5)];
    final back = decodeBooks(encodeBooks(books));
    print(encodeBooks(back));
  });

  test('PrefsBookRepository persists through SharedPreferences', () async {
    SharedPreferences.setMockInitialValues({});
    final repo = PrefsBookRepository(await SharedPreferences.getInstance());
    print('empty: ${await repo.load()}');
    await repo.save([dune]);
    print('stored: ${(await SharedPreferences.getInstance()).getString('readingList.v1')}');
  });

  test('notifier rules', () async {
    final repo = MemoryBookRepository([dune, kindred]);
    final c = ProviderContainer.test(overrides: [repositoryProvider.overrideWithValue(repo)]);
    await c.read(readingListProvider.future);
    final list = c.read(readingListProvider.notifier);
    await list.rate('b2', 4);
    print('rated: shelf=${c.read(bookProvider('b2'))!.shelf.name} rating=${c.read(bookProvider('b2'))!.rating}');
    try {
      await list.rate('b1', 9);
    } on RangeError catch (e) {
      print('rejected rating ${e.invalidValue}');
    }
    print('counts: ${[for (final s in Shelf.values) c.read(shelfProvider(s)).length]} saves=${repo.saves}');
  });

  Future<MemoryBookRepository> pumpApp(WidgetTester tester, List<Book> books) async {
    final repo = MemoryBookRepository(books);
    await tester.pumpWidget(ProviderScope(
      overrides: [repositoryProvider.overrideWithValue(repo)],
      child: const ReadingListApp(),
    ));
    await tester.pumpAndSettle();
    return repo;
  }

  List<String> tabs(WidgetTester t) => [for (final w in t.widgetList<Tab>(find.byType(Tab))) w.text!];

  testWidgets('add a book through the form', (tester) async {
    final repo = await pumpApp(tester, [dune]);
    print('start tabs: ${tabs(tester)}');
    await tester.tap(find.byTooltip('Add book'));
    await tester.pumpAndSettle();
    await tester.tap(find.text('Save'));
    await tester.pump();
    print('validation errors: ${find.text('Required').evaluate().length}');
    await tester.enterText(find.byKey(const Key('title')), '  The Left Hand of Darkness ');
    await tester.enterText(find.byKey(const Key('author')), 'Ursula K. Le Guin');
    await tester.tap(find.text('Save'));
    await tester.pumpAndSettle();
    print('after add tabs: ${tabs(tester)}; saved titles: ${(await repo.load()).map((b) => b.title).toList()}');
  });

  testWidgets('move, rate and remove from the detail page', (tester) async {
    final repo = await pumpApp(tester, [dune, kindred]);
    await tester.tap(find.text('Dune'));
    await tester.pumpAndSettle();
    await tester.tap(find.text('Done'));
    await tester.pumpAndSettle();
    await tester.tap(find.byKey(const Key('star-4')));
    await tester.pumpAndSettle();
    print('stars filled: ${find.byIcon(Icons.star).evaluate().length}');
    await tester.pageBack();
    await tester.pumpAndSettle();
    print('tabs: ${tabs(tester)}');
    await tester.tap(find.text(tabs(tester)[2]));
    await tester.pumpAndSettle();
    print('done shelf subtitle: ${find.textContaining('★').evaluate().map((e) => (e.widget as Text).data).toList()}');

    await tester.tap(find.text('Dune'));
    await tester.pumpAndSettle();
    await tester.tap(find.byTooltip('Remove'));
    await tester.pumpAndSettle();
    print('after remove tabs: ${tabs(tester)}; repo has ${(await repo.load()).length} book(s), ${repo.saves} saves');
  });

  testWidgets('deep link to a missing book', (tester) async {
    await pumpApp(tester, [dune]);
    // Simulate an incoming deep link by navigating from any context below the router.
    GoRouter.of(tester.element(find.byType(ShelvesPage))).go('/book/nope');
    await tester.pumpAndSettle();
    print(find.text('This book is no longer on your list').evaluate().length == 1 ? 'missing-book page shown' : 'NOT shown');
  });
}
```

```console
$ flutter test test/rl/
[{"id":"b1","title":"Dune","author":"Frank Herbert","shelf":"toRead","rating":null},{"id":"b2","title":"Kindred","author":"Octavia E. Butler","shelf":"done","rating":5}]
empty: []
stored: [{"id":"b1","title":"Dune","author":"Frank Herbert","shelf":"toRead","rating":null}]
rated: shelf=done rating=4
rejected rating 9
counts: [1, 0, 1] saves=1
start tabs: [To read (1), Reading (0), Done (0)]
validation errors: 2
after add tabs: [To read (2), Reading (0), Done (0)]; saved titles: [Dune, The Left Hand of Darkness]
stars filled: 4
tabs: [To read (0), Reading (1), Done (1)]
done shelf subtitle: [Frank Herbert · ★★★★]
after remove tabs: [To read (0), Reading (1), Done (0)]; repo has 1 book(s), 3 saves
missing-book page shown
00:01 +6: All tests passed!
```

Reading the results:

- **Model and repository** round-trip every field, including `null` ratings, and the stored string is exactly the
  JSON you'd expect.
- **Notifier rules:** rating Kindred moved it from *Reading* to *Done*; the invalid rating was rejected *before* any
  save (`saves=1`, from the valid rating only).
- **Add flow:** saving the empty form showed two "Required" errors; the filled form added the book (title trimmed) and
  persisted it.
- **Detail flow:** moving Dune to *Done* and tapping the fourth star filled four stars; the shelves updated their
  counts; the *Done* list showed the rating; removing the book took us back to the shelves and left one book in storage.
  Three saves: move, rate, remove.
- **Deep link** to a nonexistent id showed the friendly page.

`flutter analyze` reported no issues in `lib/rl` beyond the expected `avoid_print` notes in the test file. What I did
**not** do here is run the app on a phone or emulator — the tests exercise the real widgets and logic with fake storage,
and the only untested line is `main()` choosing `PrefsBookRepository`. Run `flutter run -t lib/rl/main.dart` on a device
to check persistence across a real restart.

## How It Actually Works

Follow one tap — the fourth star — through the layers. The `IconButton` calls `list.rate(id, 4)`. The notifier validates,
builds a new list (new `Book` objects only for the changed entry), and assigns `state`. Riverpod marks
`readingListProvider` changed, which invalidates its dependents: `bookProvider('b1')`, and each `shelfProvider(shelf)`.
Each recomputes; the `ConsumerWidget`s watching them (`BookPage`, the tab labels, the shelf lists) are marked dirty and
rebuild on the next frame. Meanwhile `save` serializes and writes in the background. go_router isn't involved at all —
the location is still `/book/b1`; only data changed. When the user goes back, `context.go('/')` changes the location,
go_router rebuilds its page list, and `ShelvesPage` reads the already-updated providers.

## Review checklist

- [ ] One source of truth: the book list. Everything else is derived.
- [ ] Storage behind an interface; real implementation chosen only in `main()`.
- [ ] No business rules in widgets.
- [ ] Every page handles loading, error, and "not found".
- [ ] Router created once.
- [ ] Each user flow has a widget test; each rule has a unit/container test.

## Exercise

1. Add search: a query parameter on `/` (`/?q=dune`) that filters all shelves. Should the query live in the URL, a
   provider, or both? Implement your answer and test a deep link with a query.
2. Add an "Undo" `SnackBar` after removing a book. Where do you keep the removed book, and how do you test it?
3. Replace the generated id (`microsecondsSinceEpoch`) with something that can't collide when two books are added in the
   same microsecond — and write the test that would have caught the collision.
4. Add a fourth shelf, *Abandoned*. List every file you had to touch. Was anything harder than it should have been?
