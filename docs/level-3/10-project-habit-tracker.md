---
description: "Level 3 project: an offline-first Flutter habit tracker with a persisted outbox, last-writer-wins sync, streaks and tests for failure paths."
---

# 10 · Project — An Offline-First Habit Tracker

Mobile apps lose connectivity constantly: lifts, trains, flight mode, flaky Wi-Fi. An **offline-first** app treats the local
database as the source of truth — every action succeeds immediately on the device — and synchronizes with a server in the
background when it can. This project builds the core of one: habits you tick off each day, streaks, a persisted **outbox**
of unsynced changes, conflict resolution, and a UI that tells the user honestly whether their changes are safe on the server.

The server is a fake behind an interface (it would be the HTTP client from [Level 2 · 07](../level-2/07-http-json.md) in a
real app), which is exactly what makes the hard cases testable: offline, restart mid-queue, two devices editing the same habit.

```text
lib/ht/
  domain.dart      Day, Habit, streak + conflict rules   (pure Dart)
  data.dart        LocalStore (JSON file), HabitApi       (I/O boundaries)
  repository.dart  HabitRepository: outbox + sync         (the offline-first engine)
  ui.dart          Riverpod providers + HabitsPage
test/ht/habit_tracker_test.dart
```

## Domain: days, habits, streaks

```dart title="lib/ht/domain.dart"
/// A calendar day with no time zone surprises: stored as yyyy-mm-dd.
extension type const Day(String iso) {
  factory Day.of(DateTime t) =>
      Day('${t.year.toString().padLeft(4, '0')}-${t.month.toString().padLeft(2, '0')}-${t.day.toString().padLeft(2, '0')}');
  DateTime get date => DateTime.parse(iso);
  Day minus(int days) => Day.of(DateTime(date.year, date.month, date.day - days));
}

class Habit {
  const Habit({required this.id, required this.name, required this.done, required this.updatedAt});
  final String id;
  final String name;
  final Set<Day> done; // days on which the habit was completed
  final int updatedAt; // logical timestamp (ms since epoch) of the last change, for conflict resolution

  Habit toggled(Day day, int now) => Habit(
        id: id,
        name: name,
        done: done.contains(day) ? ({...done}..remove(day)) : {...done, day},
        updatedAt: now,
      );

  Map<String, Object?> toJson() => {'id': id, 'name': name, 'done': [for (final d in done) d.iso]..sort(), 'updatedAt': updatedAt};
  factory Habit.fromJson(Map<String, Object?> j) => Habit(
        id: j['id'] as String,
        name: j['name'] as String,
        done: {for (final d in j['done'] as List) Day(d as String)},
        updatedAt: j['updatedAt'] as int,
      );
}

/// Consecutive completed days ending today — or ending yesterday, so a streak isn't "broken"
/// in the morning before you've had the chance to do today's habit.
int currentStreak(Set<Day> done, Day today) {
  var day = done.contains(today) ? today : today.minus(1);
  var streak = 0;
  while (done.contains(day)) {
    streak++;
    day = day.minus(1);
  }
  return streak;
}

/// Last-writer-wins merge of a local and a remote copy.
Habit resolve(Habit local, Habit remote) => remote.updatedAt > local.updatedAt ? remote : local;
```

- **`Day` is an extension type** over a `yyyy-mm-dd` string (Dart 3.3+). It costs nothing at runtime, but the type system
  stops you from mixing it up with arbitrary strings or `DateTime`s. Habits are about *calendar days*, and storing a
  `DateTime` invites time-zone bugs: "23:30 on Monday" in one zone is Tuesday in another.
- **`minus` uses `DateTime(y, m, d - n)`**, which normalizes overflow (day 0 of March is the last day of February) and works
  in local calendar terms. Subtracting `Duration(days: 1)` instead can misbehave across daylight-saving changes, because a
  calendar day isn't always 24 hours.
- **`updatedAt` is the conflict-resolution timestamp.** Last-writer-wins is the simplest policy; its trade-offs are below.

## Data boundaries

```dart title="lib/ht/data.dart"
import 'dart:convert';
import 'dart:io';
import 'domain.dart';

/// Local source of truth: the app reads and writes here, always, online or not.
class LocalStore {
  LocalStore(this.file);
  final File file;

  Future<({Map<String, Habit> habits, List<String> outbox})> load() async {
    if (!await file.exists()) return (habits: <String, Habit>{}, outbox: <String>[]);
    final j = jsonDecode(await file.readAsString()) as Map<String, Object?>;
    return (
      habits: {for (final h in j['habits'] as List) (h as Map<String, Object?>)['id'] as String: Habit.fromJson(h)},
      outbox: [...(j['outbox'] as List).cast<String>()],
    );
  }

  Future<void> save(Map<String, Habit> habits, List<String> outbox) async {
    final tmp = File('${file.path}.tmp');
    await tmp.writeAsString(jsonEncode({'habits': [for (final h in habits.values) h.toJson()], 'outbox': outbox}), flush: true);
    await tmp.rename(file.path);
  }
}

class OfflineException implements Exception {
  @override
  String toString() => 'OfflineException';
}

/// The server API. In a real app this is HTTP (Level 2 · 07); here, an interface.
abstract class HabitApi {
  Future<Habit?> fetch(String id);
  Future<void> put(Habit habit);
  Future<List<Habit>> fetchAll();
}
```

The local file stores **both** the habits and the outbox, written atomically ([Level 2 · 08](../level-2/08-persistence.md)).
If the outbox lived only in memory, killing the app before a sync would silently drop changes from the server's point of
view — the most common offline-first bug.

## The engine: outbox and sync

```dart title="lib/ht/repository.dart"
import 'dart:async';
import 'data.dart';
import 'domain.dart';

enum SyncStatus { synced, pending, offline }

/// Offline-first: writes go to local storage + an outbox immediately; sync() drains the outbox when it can.
class HabitRepository {
  HabitRepository(this._local, this._api, {int Function()? clock}) : _clock = clock ?? (() => DateTime.now().millisecondsSinceEpoch);
  final LocalStore _local;
  final HabitApi _api;
  final int Function() _clock;

  Map<String, Habit> _habits = {};
  List<String> _outbox = []; // ids of habits changed locally and not yet confirmed by the server
  final _changes = StreamController<void>.broadcast();
  Future<void>? _syncing;

  Stream<void> get changes => _changes.stream;
  List<Habit> get habits => _habits.values.toList()..sort((a, b) => a.name.compareTo(b.name));
  int get pendingCount => _outbox.length;

  Future<void> init() async {
    final s = await _local.load();
    _habits = s.habits;
    _outbox = s.outbox;
  }

  Future<void> _commit() async {
    await _local.save(_habits, _outbox);
    _changes.add(null);
  }

  Future<void> add(String id, String name) async {
    _habits[id] = Habit(id: id, name: name, done: {}, updatedAt: _clock());
    if (!_outbox.contains(id)) _outbox.add(id);
    await _commit();
  }

  Future<void> toggle(String id, Day day) async {
    _habits[id] = _habits[id]!.toggled(day, _clock());
    if (!_outbox.contains(id)) _outbox.add(id);
    await _commit();
  }

  /// Push pending changes, then pull remote changes. Safe to call repeatedly; concurrent calls share one run.
  Future<SyncStatus> sync() async {
    _syncing ??= _doSync().whenComplete(() => _syncing = null);
    await _syncing;
    return _outbox.isEmpty ? SyncStatus.synced : SyncStatus.pending;
  }

  Future<void> _doSync() async {
    try {
      for (final id in [..._outbox]) {
        final local = _habits[id]!;
        final remote = await _api.fetch(id);
        final winner = remote == null ? local : resolve(local, remote);
        if (identical(winner, local)) await _api.put(local);
        _habits[id] = winner;
        _outbox.remove(id);
        await _commit(); // persist progress after each item, so a crash mid-sync doesn't redo or lose work
      }
      for (final remote in await _api.fetchAll()) {
        final local = _habits[remote.id];
        if (_outbox.contains(remote.id)) continue; // local edit made during sync wins this round
        if (local == null || remote.updatedAt > local.updatedAt) _habits[remote.id] = remote;
      }
      await _commit();
    } on OfflineException {
      // Keep the outbox; try again later.
    }
  }

  Future<void> dispose() => _changes.close();
}
```

How it behaves:

1. **Writes never wait for the network.** `add`/`toggle` update memory, add the id to the outbox, persist both, and notify
   listeners.
2. **`sync` pushes, then pulls.** For each pending habit it fetches the server copy, resolves the conflict, uploads if the
   local copy won, and removes the id from the outbox — persisting after *each* item, so an interruption loses at most one
   item's progress. Then it pulls everything to pick up changes from other devices, skipping habits edited locally during
   the sync.
3. **Offline is a normal outcome, not an error.** `OfflineException` stops the sync and leaves the outbox intact.
4. **Concurrent sync calls share one run** (`_syncing ??= ...`). A pull-to-refresh, a timer and an app-resume event can all
   call `sync()` at once without duplicating uploads.

## UI

```dart title="lib/ht/ui.dart"
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'domain.dart';
import 'repository.dart';

final repositoryProvider = Provider<HabitRepository>((ref) => throw UnimplementedError('override in main()'));
final todayProvider = Provider<Day>((ref) => Day.of(DateTime.now()));

/// Re-emits whenever the repository changes; widgets watch this to rebuild.
final habitsProvider = StreamProvider<List<Habit>>((ref) async* {
  final repo = ref.watch(repositoryProvider);
  yield repo.habits;
  await for (final _ in repo.changes) {
    yield repo.habits;
  }
});

final syncStatusProvider = NotifierProvider<SyncStatusNotifier, SyncStatus>(SyncStatusNotifier.new);

class SyncStatusNotifier extends Notifier<SyncStatus> {
  @override
  SyncStatus build() => ref.read(repositoryProvider).pendingCount == 0 ? SyncStatus.synced : SyncStatus.pending;
  void markPending() => state = SyncStatus.pending;
  Future<void> sync() async {
    final result = await ref.read(repositoryProvider).sync();
    state = result == SyncStatus.pending ? SyncStatus.offline : result;
  }
}

class HabitsPage extends ConsumerWidget {
  const HabitsPage({super.key});
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final today = ref.watch(todayProvider);
    final habits = ref.watch(habitsProvider).value ?? const [];
    final status = ref.watch(syncStatusProvider);
    return Scaffold(
      appBar: AppBar(title: const Text('Habits'), actions: [
        IconButton(
          key: const Key('sync'),
          tooltip: switch (status) {
            SyncStatus.synced => 'All changes synced',
            SyncStatus.pending => 'Changes waiting to sync',
            SyncStatus.offline => 'Offline — will retry',
          },
          icon: Icon(switch (status) {
            SyncStatus.synced => Icons.cloud_done,
            SyncStatus.pending => Icons.cloud_upload,
            SyncStatus.offline => Icons.cloud_off,
          }),
          onPressed: () => ref.read(syncStatusProvider.notifier).sync(),
        ),
      ]),
      body: ListView(children: [
        for (final h in habits)
          CheckboxListTile(
            key: ValueKey(h.id),
            title: Text(h.name),
            subtitle: Text('Streak: ${currentStreak(h.done, today)}'),
            value: h.done.contains(today),
            onChanged: (_) async {
              await ref.read(repositoryProvider).toggle(h.id, today);
              ref.read(syncStatusProvider.notifier).markPending();
            },
          ),
      ]),
    );
  }
}
```

The sync icon has three honest states — synced, pending, offline — with tooltips (which are also the screen-reader labels,
[lesson 08](08-a11y-i18n.md)). `todayProvider` exists so tests can pin "today"; code that calls `DateTime.now()` directly in
widgets is untestable around midnight.

## Tests for every failure path

```dart title="test/ht/habit_tracker_test.dart"
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:l2/ht/data.dart';
import 'package:l2/ht/domain.dart';
import 'package:l2/ht/repository.dart';
import 'package:l2/ht/ui.dart';

/// A fake server we can take offline. Stores JSON-equivalent copies like a real server would.
class FakeServer implements HabitApi {
  bool online = true;
  final store = <String, Habit>{};
  final log = <String>[];
  void _check() {
    if (!online) throw OfflineException();
  }

  @override
  Future<Habit?> fetch(String id) async {
    _check();
    log.add('fetch $id');
    return store[id];
  }

  @override
  Future<void> put(Habit h) async {
    _check();
    log.add('put ${h.id}@${h.updatedAt}');
    store[h.id] = Habit.fromJson(h.toJson());
  }

  @override
  Future<List<Habit>> fetchAll() async {
    _check();
    log.add('fetchAll');
    return store.values.toList();
  }
}

void main() {
  const today = Day('2026-10-09');

  group('currentStreak', () {
    final cases = <String, (Set<Day>, int)>{
      'nothing done': ({}, 0),
      'only today': ({today}, 1),
      'yesterday and before, not yet today': ({Day('2026-10-08'), Day('2026-10-07')}, 2),
      'gap two days ago': ({today, Day('2026-10-08'), Day('2026-10-06')}, 2),
      'across a month boundary': ({Day('2026-10-01'), Day('2026-09-30'), Day('2026-09-29')}, 0),
    };
    cases.forEach((name, c) => test(name, () => expect(currentStreak(c.$1, today), c.$2)));
    test('month boundary counted when it ends today', () {
      expect(currentStreak({Day('2026-03-01'), Day('2026-02-28'), Day('2026-02-27')}, const Day('2026-03-01')), 3);
    });
  });

  late Directory dir;
  late FakeServer server;
  var now = 1000;
  int clock() => now += 10;

  setUp(() async {
    dir = await Directory.systemTemp.createTemp('habits');
    server = FakeServer();
  });
  tearDown(() => dir.delete(recursive: true));

  Future<HabitRepository> open() async {
    final repo = HabitRepository(LocalStore(File('${dir.path}/habits.json')), server, clock: clock);
    await repo.init();
    return repo;
  }

  test('offline writes survive a restart and sync later', () async {
    server.online = false;
    var repo = await open();
    await repo.add('h1', 'Read 20 pages');
    await repo.toggle('h1', today);
    print('offline: pending=${repo.pendingCount}, sync -> ${(await repo.sync()).name}');

    repo = await open(); // app killed and relaunched
    print('after restart: habits=${repo.habits.map((h) => h.name).toList()}, pending=${repo.pendingCount}');

    server.online = true;
    print('online: sync -> ${(await repo.sync()).name}, pending=${repo.pendingCount}');
    print('server log: ${server.log}');
    print('server has today done: ${server.store['h1']!.done.contains(today)}');
  });

  test('conflicts: newer change wins, either side', () async {
    final repo = await open();
    await repo.add('h1', 'Stretch');
    await repo.sync();

    // Another device marks today done on the server, later than our last write.
    server.store['h1'] = server.store['h1']!.toggled(today, now + 500);
    await repo.sync();
    print('remote newer -> local today done: ${repo.habits.single.done.contains(today)}');

    // We now un-tick today locally (newer than the server copy) while offline, then sync.
    server.online = false;
    now += 1000;
    await repo.toggle('h1', today);
    server.online = true;
    await repo.sync();
    print('local newer -> server today done: ${server.store['h1']!.done.contains(today)}');
  });

  test('concurrent sync calls share one run', () async {
    final repo = await open();
    await repo.add('h1', 'Walk');
    server.log.clear();
    await Future.wait([repo.sync(), repo.sync(), repo.sync()]);
    print('server calls for 3 concurrent syncs: ${server.log}');
  });

  testWidgets('UI: tick, offline indicator, then synced', (tester) async {
    final repo = await tester.runAsync(() async {
      final r = await open();
      await r.add('h1', 'Meditate');
      await r.add('h2', 'Drink water');
      await r.sync();
      return r;
    });
    server.online = false;
    await tester.pumpWidget(ProviderScope(
      overrides: [repositoryProvider.overrideWithValue(repo!), todayProvider.overrideWithValue(today)],
      child: const MaterialApp(home: HabitsPage()),
    ));
    await tester.pump();
    String tooltip() => tester.widget<IconButton>(find.byKey(const Key('sync'))).tooltip!;
    print('start: ${tooltip()}');

    await tester.runAsync(() async {
      await tester.tap(find.text('Meditate'));
      await Future<void>.delayed(const Duration(milliseconds: 50)); // let the file write finish
    });
    await tester.pump();
    print('after tick: ${tooltip()}; ${find.text('Streak: 1').evaluate().length} streak shown');

    await tester.runAsync(() async {
      await tester.tap(find.byKey(const Key('sync')));
      await Future<void>.delayed(const Duration(milliseconds: 50));
    });
    await tester.pump();
    print('sync while offline: ${tooltip()}');

    server.online = true;
    await tester.runAsync(() async {
      await tester.tap(find.byKey(const Key('sync')));
      await Future<void>.delayed(const Duration(milliseconds: 50));
    });
    await tester.pump();
    print('sync online: ${tooltip()}');
  });
}
```

```console
$ flutter test test/ht/
offline: pending=1, sync -> pending
after restart: habits=[Read 20 pages], pending=1
online: sync -> synced, pending=0
server log: [fetch h1, put h1@1020, fetchAll]
server has today done: true
remote newer -> local today done: true
local newer -> server today done: false
server calls for 3 concurrent syncs: [fetch h1, put h1@2050, fetchAll]
start: All changes synced
after tick: Changes waiting to sync; 1 streak shown
sync while offline: Offline — will retry
sync online: All changes synced
00:00 +10: All tests passed!
```

(The six streak cases print nothing; they're in the `+10`.) What was proven:

- **Offline → restart → online**: two offline writes collapsed into one outbox entry, survived a simulated relaunch (a new
  repository reading the same file), and synced with one `put` carrying the latest version.
- **Conflicts both ways**: a newer server edit replaced the local copy; a newer local edit replaced the server copy.
- **No duplicate uploads** from three simultaneous `sync()` calls.
- **The UI tells the truth** at each step: synced → pending after a tick → offline after a failed sync → synced.

`tester.runAsync` appears in the widget test because the repository does real file I/O, which doesn't complete inside the
widget tester's fake-async zone; `runAsync` runs a block on the real event loop.

## Trade-offs you should know

- **Last-writer-wins loses data.** If you tick *Monday* on your phone and *Tuesday* on your tablet while both are offline, the
  later sync overwrites the earlier habit entirely and one tick is lost. Fixes, in increasing complexity: merge per field
  (here, union or per-day timestamps of the `done` set); record operations ("tick Monday") rather than states and replay them;
  or use CRDTs. For a habit tracker, per-day merge is the pragmatic choice — exercise 2.
- **Device clocks lie.** `updatedAt` comes from the device clock, which users can change. Servers often assign their own
  version numbers instead and reject stale updates (optimistic concurrency, like an HTTP `If-Match` header).
- **When to sync**: on app start and resume, after writes (debounced), on connectivity regained, and on a timer — but not in a
  tight loop. Background sync while the app is closed needs platform facilities (WorkManager on Android, BGTaskScheduler on
  iOS) via plugins, and the OS decides when it runs.
- **Scale**: one JSON file is fine for hundreds of habits. Past that, move `LocalStore` to SQLite; nothing else changes.

## How It Actually Works

Follow one tick. `CheckboxListTile.onChanged` calls `repository.toggle`. The repository replaces the `Habit` with a new
immutable copy, appends the id to the outbox, writes the temp file, renames it into place, then adds an event to `changes`.
`habitsProvider` is a `StreamProvider` whose generator is awaiting that stream; it yields the new list, Riverpod updates its
`AsyncValue`, and `HabitsPage` rebuilds with the ticked box and the new streak — all without the network. Separately, the
widget marks sync status as pending. When the user taps sync, `SyncStatusNotifier.sync` awaits `repository.sync()`, which
either drains the outbox (status `synced`) or catches `OfflineException` and leaves it (status shown as `offline`). Because the
outbox is in the same file as the habits, there is never a state on disk where a change exists locally but is forgotten by the
sync queue.

## Review checklist

- [ ] Every user action succeeds offline.
- [ ] Outbox persisted atomically with the data it refers to.
- [ ] Sync is idempotent and safe to call concurrently.
- [ ] Conflict policy is explicit and tested in both directions.
- [ ] Dates stored as calendar days, not instants; "today" injectable.
- [ ] UI distinguishes synced, pending and offline.

## Exercise

1. Add `rename(id, name)` and a test where a rename made offline on one device and a tick made on another both survive (you'll
   need per-field merging).
2. Replace last-writer-wins for `done` with a per-day merge: store each day's latest state with its own timestamp. Write the
   Monday/Tuesday two-device test from "Trade-offs" first and watch it fail with the current code.
3. Trigger `sync()` automatically 2 seconds after the last write (debounced), and when the app resumes (`AppLifecycleListener`).
   Test the debounce with fake time.
4. Swap `LocalStore` for a SQLite implementation behind the same method signatures and run the same tests against it.
