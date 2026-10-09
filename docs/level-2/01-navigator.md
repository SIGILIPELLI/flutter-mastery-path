---
description: "Flutter's Navigator in practice: push and pop as a stack, awaiting a result from a page, PopScope for unsaved changes, and tests that follow every route."
---

# 01 · Navigation: push, pop & Returning Results

Apps have more than one screen. Flutter's `Navigator` manages them as a **stack of routes**: push a
route and it slides in on top; pop it and the one beneath is visible again. That model is simple,
and it has one feature people underuse — `push` returns a `Future` that completes with whatever the
pushed page pops with, so a page can *return a result*. This lesson builds a contacts list with an
edit page that returns the new name, guards unsaved edits with `PopScope`, and tests every path while
logging the route stack. The next lesson moves to URL-based routing with `go_router`.

## The pages

```dart title="navigator_demo.dart"
import 'package:flutter/material.dart';

class Contact {
  const Contact(this.id, this.name);
  final int id;
  final String name;
}

class ContactsPage extends StatefulWidget {
  const ContactsPage({super.key});
  @override
  State<ContactsPage> createState() => _ContactsPageState();
}

class _ContactsPageState extends State<ContactsPage> {
  final _contacts = [const Contact(1, 'Ada'), const Contact(2, 'Grace')];
  String? _lastMessage;

  Future<void> _edit(Contact c) async {
    // push returns a Future that completes when the pushed route is popped.
    final renamed = await Navigator.of(context).push<String>(
      MaterialPageRoute(builder: (_) => EditContactPage(contact: c)),
    );
    if (!mounted) return; // the user may have left this page meanwhile
    setState(() {
      if (renamed == null) {
        _lastMessage = 'Edit of ${c.name} cancelled';
      } else {
        final i = _contacts.indexWhere((x) => x.id == c.id);
        _contacts[i] = Contact(c.id, renamed);
        _lastMessage = 'Renamed to $renamed';
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Contacts')),
      body: Column(
        children: [
          for (final c in _contacts) ListTile(title: Text(c.name), onTap: () => _edit(c)),
          if (_lastMessage != null) Text(_lastMessage!, key: const Key('message')),
        ],
      ),
    );
  }
}

class EditContactPage extends StatefulWidget {
  const EditContactPage({super.key, required this.contact});
  final Contact contact;
  @override
  State<EditContactPage> createState() => _EditContactPageState();
}

class _EditContactPageState extends State<EditContactPage> {
  late final _name = TextEditingController(text: widget.contact.name);
  bool get _dirty => _name.text != widget.contact.name;

  @override
  void initState() {
    super.initState();
    _name.addListener(() => setState(() {}));
  }

  @override
  void dispose() {
    _name.dispose();
    super.dispose();
  }

  Future<bool> _confirmDiscard() async {
    final discard = await showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Discard changes?'),
        actions: [
          TextButton(onPressed: () => Navigator.pop(context, false), child: const Text('Keep editing')),
          TextButton(onPressed: () => Navigator.pop(context, true), child: const Text('Discard')),
        ],
      ),
    );
    return discard ?? false;
  }

  @override
  Widget build(BuildContext context) {
    // Intercepts back (system back, AppBar back) while there are unsaved edits.
    return PopScope<String>(
      canPop: !_dirty,
      onPopInvokedWithResult: (didPop, result) async {
        if (didPop) return;
        if (await _confirmDiscard() && context.mounted) Navigator.pop(context); // pops with null
      },
      child: Scaffold(
        appBar: AppBar(title: Text('Edit ${widget.contact.name}')),
        body: Padding(
          padding: const EdgeInsets.all(16),
          child: Column(children: [
            TextField(key: const Key('name'), controller: _name),
            FilledButton(
              onPressed: _dirty ? () => Navigator.pop(context, _name.text.trim()) : null,
              child: const Text('Save'),
            ),
          ]),
        ),
      ),
    );
  }
}
```

The flow:

1. `ContactsPage._edit` calls `Navigator.of(context).push<String>(MaterialPageRoute(...))` and
   **awaits** it.
2. `EditContactPage` ends either with `Navigator.pop(context, newName)` (Save) or a plain pop (back),
   which completes the future with `null`.
3. Back on the list, `if (!mounted) return;` guards the `setState`: between the `await` and its result,
   the user could have left this page entirely.
4. `PopScope` sits around the edit page. While the text differs from the original (`_dirty`),
   `canPop: false` blocks the pop; `onPopInvokedWithResult` is still called (with `didPop: false`), so the
   page can ask "Discard changes?" and pop itself if the user agrees.

`MaterialPageRoute` gives the platform-appropriate transition and back gesture; `showDialog` pushes a
`DialogRoute` onto the *same* navigator — dialogs are routes too, and `Navigator.pop(context, true)` inside
the dialog returns `true` to the `await showDialog(...)`.

## Testing every path

```dart title="navigator_test.dart"
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:l2/n/navigator_demo.dart';

class RouteLog extends NavigatorObserver {
  final events = <String>[];
  String _name(Route<dynamic>? r) => r is MaterialPageRoute ? 'page' : r is DialogRoute ? 'dialog' : '${r?.runtimeType}';
  @override
  void didPush(Route route, Route? previous) => events.add('push ${_name(route)}');
  @override
  void didPop(Route route, Route? previous) => events.add('pop ${_name(route)}');
}

void main() {
  late RouteLog log;
  Future<void> start(WidgetTester tester) async {
    log = RouteLog();
    await tester.pumpWidget(MaterialApp(home: const ContactsPage(), navigatorObservers: [log]));
  }

  testWidgets('save returns a result to the previous page', (tester) async {
    await start(tester);
    await tester.tap(find.text('Ada'));
    await tester.pumpAndSettle(); // wait for the page transition animation
    await tester.enterText(find.byKey(const Key('name')), 'Ada Lovelace');
    await tester.pump();
    await tester.tap(find.text('Save'));
    await tester.pumpAndSettle();
    print('message: ${tester.widget<Text>(find.byKey(const Key('message'))).data}');
    print('routes:  ${log.events}');
    expect(find.text('Ada Lovelace'), findsOneWidget);
  });

  testWidgets('back with no edits pops immediately with null', (tester) async {
    await start(tester);
    await tester.tap(find.text('Grace'));
    await tester.pumpAndSettle();
    await tester.pageBack();
    await tester.pumpAndSettle();
    print('message: ${tester.widget<Text>(find.byKey(const Key('message'))).data}');
  });

  testWidgets('back with edits asks first', (tester) async {
    await start(tester);
    await tester.tap(find.text('Grace'));
    await tester.pumpAndSettle();
    await tester.enterText(find.byKey(const Key('name')), 'Grace Hopper');
    await tester.pump();

    await tester.pageBack();
    await tester.pumpAndSettle();
    print('dialog shown: ${find.text('Discard changes?').evaluate().isNotEmpty}');
    await tester.tap(find.text('Keep editing'));
    await tester.pumpAndSettle();
    print('still editing: ${find.text('Edit Grace').evaluate().isNotEmpty}');

    await tester.pageBack();
    await tester.pumpAndSettle();
    await tester.tap(find.text('Discard'));
    await tester.pumpAndSettle();
    print('message: ${tester.widget<Text>(find.byKey(const Key('message'))).data}');
    print('routes:  ${log.events}');
  });
}
```

```console
$ flutter test test/n/navigator_test.dart
message: Renamed to Ada Lovelace
routes:  [push page, push page, pop page]
message: Edit of Grace cancelled
dialog shown: true
still editing: true
message: Edit of Grace cancelled
routes:  [push page, push page, push dialog, pop dialog, push dialog, pop dialog, pop page]
00:01 +3: All tests passed!
```

- The first `push page` in each log is the home route `MaterialApp` pushes at startup. Then the edit page
  is pushed, and popped with the new name, which arrived back in `_edit`.
- **Back with no edits** popped straight away, and `_edit` received `null` → "cancelled".
- **Back with edits** was blocked; the dialog appeared. "Keep editing" popped only the dialog. Back again,
  "Discard" popped the dialog and then the page.

Test helpers used here:

- `tester.pumpAndSettle()` pumps frames until no animations are running — needed after navigation, because
  page transitions animate for a few hundred milliseconds.
- `tester.pageBack()` taps the `AppBar`'s back button, the same path a user's tap takes.
- A `NavigatorObserver` passed to `MaterialApp.navigatorObservers` sees every push and pop. Analytics
  packages use the same hook to log screen views.

## When the imperative stack is enough

`push`/`pop` is direct and easy to follow, and fine for:

- flows that return a value (pick a date, edit an item, confirm a choice);
- dialogs, bottom sheets and menus (all routes);
- apps without deep links or a web version.

It gets awkward when the **URL** matters: opening `myapp://contacts/2/edit` from a notification, using the
browser's address bar and back button on the web, or restoring the stack after the OS kills the app. Those
need the stack to be *derived from a location*, which is what `go_router` (and the underlying `Router` API)
provides — [lesson 02](02-go-router.md).

## How It Actually Works

`Navigator` is a `StatefulWidget` that owns a list of `Route` objects and renders each active route's content
in an `Overlay` — a `Stack` of entries, with the top route on top. `MaterialApp` creates one for you;
`Navigator.of(context)` finds the nearest one above the given context (which is why a dialog's builder gets its
own `context`, and why `Navigator.pop(context)` inside it pops the dialog).

`push` adds a route, runs its entrance transition, and returns `route.popped` — a `Future` completed when the
route is popped, with the value passed to `pop`. Pops go through `Route.popDisposition`: when a route's
`PopScope` has `canPop: false`, a back gesture or button doesn't remove the route; the framework calls
`onPopInvokedWithResult(false, null)` instead and leaves the decision to you. That design also cooperates with
Android's **predictive back** gesture: because the framework knows in advance whether a pop will be allowed, the
system can show the preview animation only when a pop will really happen.

## Common mistakes

- **Using `context` after an `await` without checking `mounted`** (or `context.mounted` in builders).
- **Forgetting that `push` returns a result**, and passing callbacks into pages instead.
- **Calling `Navigator.pop` from the wrong context** — popping the page when you meant the dialog.
- **`WillPopScope`** in old tutorials: it's deprecated in favour of `PopScope`.
- **Not using `pumpAndSettle`** in navigation tests, then asserting during a transition.

## Exercise

1. Add an "Add contact" button that pushes the edit page with an empty name and appends the result.
2. Show the rename confirmation as a `SnackBar` instead of a `Text`. Where must `ScaffoldMessenger.of` be called
   so the snackbar survives the pop?
3. Use `Navigator.pushReplacement` for a "Save and edit next" button. What does the route log show?
4. Write a test for the race condition: start the edit, then (in the test) replace the whole app before popping.
   Confirm no `setState after dispose` error thanks to the `mounted` check.
