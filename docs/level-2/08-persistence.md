---
description: "Store data on the device in Flutter: shared_preferences for settings, JSON files with atomic writes and schema migration, and when to use SQLite."
---

# 08 · Local Persistence

Anything in memory disappears when the OS kills your app — which on mobile happens routinely, not just when the
user swipes it away. Picking the right storage is mostly about the *shape* of the data:

| Data | Good fit |
|---|---|
| A few settings and flags | `shared_preferences` (key–value) |
| A document or small collection you load and save whole | a JSON file in the app's documents directory |
| Many records you query, filter, sort, page | SQLite (`sqflite`, or `drift` for a typed layer) or another embedded database |
| Secrets (tokens, keys) | platform keystores via `flutter_secure_storage` |
| Large binary files (images, downloads) | files on disk, with paths stored in one of the above |

This lesson builds the first two properly and tests them; it ran with shared_preferences 2.5.6.

## Settings with shared_preferences

```bash
flutter pub add shared_preferences
```

```dart title="persistence_demo.dart (settings)"
import 'dart:convert';
import 'dart:io';
import 'package:shared_preferences/shared_preferences.dart';

/// Small settings -> key-value store.
class SettingsStore {
  SettingsStore(this._prefs);
  final SharedPreferences _prefs;

  static const _darkKey = 'settings.darkMode';
  static const _tipKey = 'settings.defaultTipPercent';

  bool get darkMode => _prefs.getBool(_darkKey) ?? false;
  int get defaultTipPercent => _prefs.getInt(_tipKey) ?? 15;

  Future<void> setDarkMode(bool v) => _prefs.setBool(_darkKey, v);
  Future<void> setDefaultTipPercent(int v) {
    if (v < 0 || v > 100) throw RangeError.range(v, 0, 100, 'percent');
    return _prefs.setInt(_tipKey, v);
  }
}

```

Wrapping the package in a `SettingsStore` keeps key names in one place (typos in string keys are silent bugs),
supplies defaults, validates values, and gives tests a seam.

shared_preferences now offers three APIs: the original `SharedPreferences` (used here — it caches everything in
memory, so reads are synchronous), `SharedPreferencesAsync` (every read goes to the platform, nothing cached), and
`SharedPreferencesWithCache` (cached, with an allow-list of keys). The package docs recommend the newer two for new
code; the wrapper pattern above works with any of them, and switching later only touches this one class.

## Documents in a JSON file

```dart title="persistence_demo.dart (notes file)"
/// Structured documents -> a JSON file, written atomically.
class Note {
  Note(this.id, this.text, this.updatedAt);
  final int id;
  final String text;
  final DateTime updatedAt;
  Map<String, dynamic> toJson() => {'id': id, 'text': text, 'updatedAt': updatedAt.toIso8601String()};
  factory Note.fromJson(Map<String, dynamic> j) =>
      Note(j['id'] as int, j['text'] as String, DateTime.parse(j['updatedAt'] as String));
}

class NotesFile {
  NotesFile(this.file);
  final File file;
  static const schemaVersion = 2;

  Future<List<Note>> load() async {
    if (!await file.exists()) return [];
    final decoded = jsonDecode(await file.readAsString()) as Map<String, dynamic>;
    final migrated = _migrate(decoded);
    return [for (final n in migrated['notes'] as List) Note.fromJson(n as Map<String, dynamic>)];
  }

  Future<void> save(List<Note> notes) async {
    final tmp = File('${file.path}.tmp');
    await tmp.writeAsString(jsonEncode({'version': schemaVersion, 'notes': [for (final n in notes) n.toJson()]}), flush: true);
    await tmp.rename(file.path); // atomic replace on the same filesystem
  }

  /// v1 stored a bare list of {id, text}; v2 wraps it and adds updatedAt.
  Map<String, dynamic> _migrate(Object? data) {
    if (data is Map<String, dynamic> && data['version'] == schemaVersion) return data;
    if (data is Map<String, dynamic> && data['version'] == 1) {
      return {
        'version': 2,
        'notes': [
          for (final n in data['notes'] as List)
            {...n as Map<String, dynamic>, 'updatedAt': DateTime.utc(2000).toIso8601String()}
        ],
      };
    }
    throw FormatException('Unknown notes file version: ${data is Map ? data['version'] : data.runtimeType}');
  }
}
```

Two habits here prevent real data loss:

- **Atomic save.** Writing directly to `notes.json` and crashing (or being killed) halfway leaves a truncated,
  unparseable file. Writing a temp file, flushing it, and renaming it over the original means the file is always
  either the old version or the new one.
- **A schema version and migrations.** The file records `version`. Old files are upgraded on load; files from a
  *newer* app version are refused instead of being half-read and then overwritten.

In the app, put the file in the documents directory from `path_provider`:

```dart
final dir = await getApplicationDocumentsDirectory(); // package:path_provider
final notes = NotesFile(File('${dir.path}/notes.json'));
```

## Tests

```dart title="persistence_test.dart"
import 'dart:convert';
import 'dart:io';
import 'package:flutter_test/flutter_test.dart';
import 'package:l2/s/persistence_demo.dart';
import 'package:shared_preferences/shared_preferences.dart';

void main() {
  test('SettingsStore: defaults, writes, and values surviving a "restart"', () async {
    SharedPreferences.setMockInitialValues({}); // in-memory fake platform store
    var store = SettingsStore(await SharedPreferences.getInstance());
    print('fresh install: dark=${store.darkMode} tip=${store.defaultTipPercent}');
    await store.setDarkMode(true);
    await store.setDefaultTipPercent(20);
    try {
      await store.setDefaultTipPercent(250);
    } on RangeError catch (e) {
      print('rejected: ${e.message} ${e.invalidValue}');
    }
    store = SettingsStore(await SharedPreferences.getInstance());
    print('after restart: dark=${store.darkMode} tip=${store.defaultTipPercent}');
  });

  test('NotesFile: round trip, atomic write, migration', () async {
    final dir = await Directory.systemTemp.createTemp('notes');
    addTearDown(() => dir.delete(recursive: true));
    final store = NotesFile(File('${dir.path}/notes.json'));

    print('missing file loads as: ${await store.load()}');
    await store.save([Note(1, 'buy milk', DateTime.utc(2026, 10, 9, 8))]);
    print('on disk: ${await store.file.readAsString()}');
    print('files in dir: ${dir.listSync().map((e) => e.uri.pathSegments.last).toList()}');
    final loaded = await store.load();
    print('loaded: ${loaded.single.text} @ ${loaded.single.updatedAt}');

    await store.file.writeAsString(jsonEncode({'version': 1, 'notes': [{'id': 7, 'text': 'old note'}]}));
    final migrated = await store.load();
    print('v1 file migrated: ${migrated.single.id} "${migrated.single.text}" ${migrated.single.updatedAt}');

    await store.file.writeAsString(jsonEncode({'version': 9, 'notes': []}));
    try {
      await store.load();
    } on FormatException catch (e) {
      print('future version: ${e.message}');
    }
  });
}
```

```console
$ flutter test test/s/persistence_test.dart
fresh install: dark=false tip=15
rejected: Invalid value 250
after restart: dark=true tip=20
missing file loads as: []
on disk: {"version":2,"notes":[{"id":1,"text":"buy milk","updatedAt":"2026-10-09T08:00:00.000Z"}]}
files in dir: [notes.json]
loaded: buy milk @ 2026-10-09 08:00:00.000Z
v1 file migrated: 7 "old note" 2000-01-01 00:00:00.000Z
future version: Unknown notes file version: 9
00:00 +2: All tests passed!
```

- `SharedPreferences.setMockInitialValues({})` replaces the platform side with an in-memory map, so the test runs
  anywhere. Be honest about what that proves: the "restart" only rebuilt `SettingsStore` on top of the same
  in-memory instance. It checks *our* logic (keys, defaults, validation), not that Android or iOS actually wrote to
  disk — that belongs in an [integration test on a device](../level-3/09-integration-tests.md).
- The file test used a real temporary directory. After `save`, only `notes.json` exists — the `.tmp` file was renamed
  away. A version-1 file was migrated with a default date, and a version-9 file was refused.

## When to use a database

JSON files load and save everything at once. When you need "the 20 most recent notes containing *milk*", or the data
grows to thousands of records, use SQLite. `sqflite` gives you raw SQL; `drift` generates typed queries and
migrations from Dart table definitions and can watch queries as streams, which fits nicely with `StreamBuilder` or
Riverpod's `StreamProvider`. The SQL itself is covered in the [SQL course](https://sigilipelli.github.io/sql-mastery-path/).
Both need a device, simulator, or desktop target to run (sqflite has a separate FFI variant for tests on desktop);
the [Level 3 project](../level-3/10-project-habit-tracker.md) builds an offline-first store with a repository
interface so the storage engine is swappable.

## How It Actually Works

`shared_preferences` is a *federated plugin*: the Dart API calls a platform implementation, which on Android uses
`SharedPreferences` (or the newer DataStore for the async API), on iOS/macOS `NSUserDefaults`, on the web
`localStorage`, and on Windows/Linux a JSON file. Those stores are designed for small values; they load whole
preference files into memory, which is why they're the wrong place for large data. The legacy `SharedPreferences`
class loads all keys once in `getInstance()` and then serves reads from that cache; writes update the cache
immediately and are persisted asynchronously.

The atomic-rename trick relies on the filesystem: `rename` within one filesystem replaces the directory entry in a
single step on POSIX systems, so readers see either the old file or the new one. `flush: true` asks the OS to push
the temp file's data out before the rename, reducing the window in which a power loss could leave an empty file.

## Common mistakes

- **Storing large lists or JSON blobs in shared_preferences.** Use a file or database.
- **Storing tokens in shared_preferences.** It's not encrypted; use the platform keystore.
- **Scattered string keys** (`prefs.getBool('darkmode')` here, `'darkMode'` there). Centralize them.
- **Non-atomic writes** and **no schema version** — fine until the first crash or the first model change.
- **Forgetting that `getApplicationDocumentsDirectory` is async** and calling it in `build`. Resolve storage once
  at startup and inject it.
- **Using `DateTime.now()` without UTC** in stored data, then comparing across time zones. Store UTC, display local.

## Exercise

1. Persist the tip splitter's default percentage from [Level 1 · 10](../level-1/10-project-tip-splitter.md) using
   `SettingsStore`, loading it before `runApp`.
2. Add a `delete(int id)` to `NotesFile` and write a test that simulates a crash: write a garbage `.tmp` file before
   calling `save` and check the result is still correct.
3. Introduce schema version 3 (add a `pinned` flag with default `false`). Migrate both v1 and v2 files, and test both.
4. Wrap `NotesFile` in a Riverpod `AsyncNotifier` that loads on `build` and saves after each change. What happens if
   two saves overlap? Fix it by serializing writes.
