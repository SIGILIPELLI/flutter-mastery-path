---
description: "Declarative URL routing in Flutter with go_router: nested routes, path and query parameters, auth redirects with refreshListenable, and error pages."
---

# 02 · Declarative Routing with go_router

In [lesson 01](01-navigator.md) the navigation stack was something you *built up* by calling `push`. That
works until the app needs to answer the question "what should be on screen for this URL?" — a deep link
from a notification, a web user pasting an address, a login redirect that should send the user back to where
they were heading. Declarative routing flips the model: you describe a **mapping from locations to pages**,
and the router derives the stack from the current location.

`go_router` is the package the Flutter team maintains for this (it lives in the `flutter/packages`
repository). This lesson was run with go_router 18.0.2; its API has had breaking majors over the years, so
check the changelog when you upgrade — the concepts below have stayed stable.

```bash
flutter pub add go_router
```

## A router for a small contacts app

The whole configuration lives in one `GoRouter` object:

```dart title="router_demo.dart"
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

/// Very small auth state; real apps use their state-management solution here.
class Session extends ChangeNotifier {
  bool loggedIn = false;
  void logIn() { loggedIn = true; notifyListeners(); }
  void logOut() { loggedIn = false; notifyListeners(); }
}

const contacts = {'1': 'Ada', '2': 'Grace', '3': 'Linus'};

GoRouter buildRouter(Session session) => GoRouter(
      initialLocation: '/contacts',
      refreshListenable: session, // re-run redirect when login state changes
      redirect: (context, state) {
        final goingToLogin = state.matchedLocation == '/login';
        if (!session.loggedIn && !goingToLogin) {
          return '/login?from=${Uri.encodeComponent(state.uri.toString())}';
        }
        if (session.loggedIn && goingToLogin) {
          return state.uri.queryParameters['from'] ?? '/contacts';
        }
        return null; // no redirect
      },
      routes: [
        GoRoute(path: '/login', builder: (context, state) => LoginPage(session: session)),
        GoRoute(
          path: '/contacts',
          builder: (context, state) => ContactListPage(filter: state.uri.queryParameters['q']),
          routes: [
            // Nested: /contacts/:id is stacked on top of /contacts
            GoRoute(
              path: ':id',
              builder: (context, state) => ContactPage(id: state.pathParameters['id']!),
            ),
          ],
        ),
      ],
      errorBuilder: (context, state) => Scaffold(
        appBar: AppBar(title: const Text('Not found')),
        body: Center(child: Text('No page at ${state.uri}')),
      ),
    );

class LoginPage extends StatelessWidget {
  const LoginPage({super.key, required this.session});
  final Session session;
  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: const Text('Log in')),
        body: Center(child: FilledButton(onPressed: session.logIn, child: const Text('Log in'))),
      );
}

class ContactListPage extends StatelessWidget {
  const ContactListPage({super.key, this.filter});
  final String? filter;
  @override
  Widget build(BuildContext context) {
    final entries = contacts.entries
        .where((e) => filter == null || e.value.toLowerCase().contains(filter!.toLowerCase()));
    return Scaffold(
      appBar: AppBar(title: Text(filter == null ? 'Contacts' : 'Contacts matching "$filter"')),
      body: ListView(children: [
        for (final e in entries)
          ListTile(title: Text(e.value), onTap: () => context.go('/contacts/${e.key}')),
      ]),
    );
  }
}

class ContactPage extends StatelessWidget {
  const ContactPage({super.key, required this.id});
  final String id;
  @override
  Widget build(BuildContext context) {
    final name = contacts[id];
    return Scaffold(
      appBar: AppBar(title: Text(name ?? 'Unknown contact')),
      body: Center(child: Text(name == null ? 'No contact $id' : 'Details for $name')),
    );
  }
}
```

Read the configuration from the outside in:

- **`routes`** is a tree. `/contacts/:id` is declared *inside* `/contacts`, so visiting `/contacts/2`
  builds **two** pages: the list underneath and the contact on top. The AppBar gets a back arrow for free,
  and back takes you to `/contacts` — even if the user arrived from a deep link and never saw the list.
- **`:id`** is a path parameter, read with `state.pathParameters['id']`. Query parameters (`?q=gr`) come from
  `state.uri.queryParameters`. Both are always strings; parse and validate them yourself.
- **`redirect`** runs before every navigation. Returning a location sends the user there; returning `null`
  lets the navigation proceed. Here, anyone logged out is sent to `/login`, carrying the original destination
  in `from`, and a logged-in user who lands on `/login` is sent on to `from`.
- **`refreshListenable: session`** makes the router re-run `redirect` whenever `session` notifies. Logging in
  doesn't navigate anywhere explicitly — the redirect does it.
- **`errorBuilder`** handles locations that match no route.

Wire it in with `MaterialApp.router`, which replaces `home`/`routes`:

```dart
final session = Session();
final router = buildRouter(session);

void main() => runApp(MaterialApp.router(routerConfig: router));
```

## `go` versus `push`

Inside the list, rows call `context.go('/contacts/${e.key}')`. `go` means *"make the location this"*; the
stack is rebuilt from the route tree. `context.push('/contacts/2')` also exists and adds a page on top of
whatever is there, like `Navigator.push`, which is handy for a page that should return a result
(`final r = await context.push<String>(...)`). Rule of thumb: use `go` for anything that represents *where the
user is*, `push` for transient, result-returning detours.

## Proving it with tests

```dart title="router_test.dart"
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:go_router/go_router.dart';
import 'package:l2/n/router_demo.dart';

void main() {
  late Session session;
  late GoRouter router;

  Future<void> start(WidgetTester tester, {bool loggedIn = true}) async {
    session = Session()..loggedIn = loggedIn;
    router = buildRouter(session);
    addTearDown(router.dispose);
    await tester.pumpWidget(MaterialApp.router(routerConfig: router));
    await tester.pumpAndSettle();
  }

  String location() => router.routerDelegate.currentConfiguration.uri.toString();
  String title(WidgetTester t) => t.widget<Text>(find.descendant(of: find.byType(AppBar), matching: find.byType(Text)).last).data!;
  int stackDepth(WidgetTester t) => t.state<NavigatorState>(find.byType(Navigator).first).widget.pages.length;

  testWidgets('a deep link builds the whole stack', (tester) async {
    await start(tester);
    router.go('/contacts/2');
    await tester.pumpAndSettle();
    print('at ${location()}: "${title(tester)}", pages in stack: ${stackDepth(tester)}');
    await tester.pageBack();
    await tester.pumpAndSettle();
    print('after back: ${location()} "${title(tester)}"');
  });

  testWidgets('query parameters', (tester) async {
    await start(tester);
    router.go('/contacts?q=gr');
    await tester.pumpAndSettle();
    print('at ${location()}: "${title(tester)}", rows: ${find.byType(ListTile).evaluate().length}');
  });

  testWidgets('logged out: redirect to login, then back to the deep link', (tester) async {
    await start(tester, loggedIn: false);
    router.go('/contacts/3');
    await tester.pumpAndSettle();
    print('asked for /contacts/3, got ${location()}');
    await tester.tap(find.widgetWithText(FilledButton, 'Log in'));
    await tester.pumpAndSettle();
    print('after login: ${location()} "${title(tester)}"');
  });

  testWidgets('unknown paths and unknown ids', (tester) async {
    await start(tester);
    router.go('/settings');
    await tester.pumpAndSettle();
    print('/settings -> "${title(tester)}": ${tester.widget<Text>(find.textContaining('No page at')).data}');
    router.go('/contacts/99');
    await tester.pumpAndSettle();
    print('/contacts/99 -> "${title(tester)}"');
  });
}
```

```console
$ flutter test test/n/router_test.dart
at /contacts/2: "Grace", pages in stack: 2
after back: /contacts "Contacts"
at /contacts?q=gr: "Contacts matching "gr"", rows: 1
asked for /contacts/3, got /login?from=%2Fcontacts%2F3
after login: /contacts/3 "Linus"
/settings -> "Not found": No page at /settings
/contacts/99 -> "Unknown contact"
00:05 +7: All tests passed!
```

(That run included lesson 01's three navigator tests too, hence `+7`.) What each line shows:

1. **Deep link → full stack.** Going straight to `/contacts/2` produced two pages, and `pageBack()` landed on
   `/contacts` — the list was built even though nobody navigated through it.
2. **Query parameters** filtered the list to one row; the title reflects the filter. Because the filter is
   in the URL, a web user can bookmark it and the browser's back button undoes it.
3. **Redirect round-trip.** Logged out, `/contacts/3` became `/login?from=%2Fcontacts%2F3` (note the encoding —
   `from` holds a URL inside a URL). Tapping "Log in" only flipped `session.loggedIn`; `refreshListenable`
   re-ran `redirect`, which saw a logged-in user on `/login` and sent them to `from`.
4. **Two kinds of "not found".** `/settings` matched no route, so `errorBuilder` ran. `/contacts/99` *did* match
   `/contacts/:id` — the router can't know contact 99 doesn't exist — so the page itself had to handle it.

## Shell routes: persistent navigation bars

Bottom navigation bars are the classic case where some UI should *stay* while the location changes beneath it.
`ShellRoute` wraps child routes in a shared scaffold; `StatefulShellRoute.indexedStack` goes further and keeps a
separate navigator (and scroll position, and stack) per tab:

```dart
StatefulShellRoute.indexedStack(
  builder: (context, state, shell) => Scaffold(
    body: shell, // the current branch's navigator
    bottomNavigationBar: NavigationBar(
      selectedIndex: shell.currentIndex,
      onDestinationSelected: shell.goBranch,
      destinations: const [
        NavigationDestination(icon: Icon(Icons.people), label: 'Contacts'),
        NavigationDestination(icon: Icon(Icons.settings), label: 'Settings'),
      ],
    ),
  ),
  branches: [
    StatefulShellBranch(routes: [GoRoute(path: '/contacts', builder: ...)]),
    StatefulShellBranch(routes: [GoRoute(path: '/settings', builder: ...)]),
  ],
)
```

Switching tabs with `goBranch` restores each tab where the user left it — the behaviour users expect from native
apps and one that's surprisingly fiddly to build by hand.

## Typed routes (optional)

String paths like `'/contacts/${e.key}'` are easy to typo. go_router has an optional code-generation companion,
`go_router_builder`, that turns route classes into type-safe `ContactRoute(id: '2').go(context)` calls. It adds a
`build_runner` step; many teams start with strings plus a few helper functions (`String contactPath(String id)`)
and adopt generation only when the route table grows.

## How It Actually Works

Flutter has two navigation layers. The imperative `Navigator` from lesson 01 is the lower layer. Above it sits the
**Router** API (sometimes called "Navigator 2.0"): a `RouteInformationProvider` reports the platform's location
(the browser URL, or an incoming deep link), a `RouteInformationParser` turns it into an app-specific
configuration, and a `RouterDelegate` builds a `Navigator` whose `pages` list is derived from that configuration.
When the configuration changes, the delegate produces a new page list and the `Navigator` diffs it against the
old one — pages with the same key are kept, new ones animate in, missing ones animate out.

go_router implements all three pieces. Its parser matches the location against your route tree and produces a
list of matches (one per nesting level, which is why `/contacts/2` yields two pages). Before building, it runs
the top-level `redirect` and then any route-level `redirect`s, looping until the location is stable (it caps the
number of redirects to catch infinite loops). `refreshListenable` simply triggers that whole pipeline again.
When the user presses back on a page, the delegate pops the last match and reports the new location upward, so
on the web the address bar stays in sync.

## Common mistakes

- **Redirect loops**: redirecting logged-out users to `/login` without excluding `/login` itself.
  go_router will stop after its redirect limit and show an error.
- **Forgetting `refreshListenable`**, then wondering why logging in doesn't move the user anywhere.
- **Using `push` for top-level destinations**, which piles pages up and breaks the browser back button.
  Use `go`.
- **Trusting path parameters**: `:id` is user-controlled text. Handle unknown ids in the page, as above.
- **Creating the `GoRouter` inside `build`**, which recreates it on every rebuild and resets navigation.
  Create it once (a top-level final, a field in a `State`, or provided by your DI).

## Exercise

1. Add `/contacts/:id/edit` as a child of `/contacts/:id`. What does the stack contain when you deep-link to it?
2. Convert the app to a `StatefulShellRoute.indexedStack` with Contacts and Settings tabs. Write a test that
   scrolls a long list in one tab, switches tabs and back, and verifies the scroll position survived.
3. Add a route-level `redirect` on `/contacts/:id` that sends non-numeric ids to `/contacts`.
4. Make the login page's "Log in" button unnecessary in tests: start with `loggedIn = false`, then call
   `session.logIn()` directly from the test. Does the user still end up at the deep link? Why?
