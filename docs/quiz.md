---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Flutter Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What makes Flutter different from wrappers around native widgets?",
    "options": [
      "It renders its own UI with its own engine, so look and behavior are consistent across platforms",
      "Uses native buttons",
      "Uses HTML",
      "Uses WebViews"
    ],
    "correctIndex": 0,
    "explanation": "Everything is a widget drawn by Flutter."
  },
  {
    "question": "What is the Flutter layout rule that explains most overflow bugs?",
    "options": [
      "Constraints go down, sizes go up, parents set positions",
      "Children choose position",
      "Sizes go down",
      "Random"
    ],
    "correctIndex": 0,
    "explanation": "Debug by following it."
  },
  {
    "question": "A Row with three long Text widgets overflows. What do you do?",
    "options": [
      "Wrap children in Expanded/Flexible (or allow wrapping/scrolling)",
      "Use setState",
      "Add padding",
      "Use Center"
    ],
    "correctIndex": 0,
    "explanation": "Row gives unbounded main-axis constraints."
  },
  {
    "question": "A counter must change when a button is pressed. What type of widget do you need?",
    "options": [
      "InheritedWidget only",
      "StatelessWidget",
      "Builder",
      "A StatefulWidget (or state from a state-management solution) calling setState"
    ],
    "correctIndex": 3,
    "explanation": "State changes trigger rebuilds."
  },
  {
    "question": "You start a Timer in a State. Where do you cancel it?",
    "options": [
      "build()",
      "In dispose()",
      "Never",
      "initState()"
    ],
    "correctIndex": 1,
    "explanation": "Avoid leaks and setState after dispose."
  },
  {
    "question": "A reorderable list of stateful tiles mixes up their text field contents. What's missing?",
    "options": [
      "Stable Keys (ValueKey/ObjectKey) on the items",
      "setState",
      "const",
      "Theme"
    ],
    "correctIndex": 0,
    "explanation": "State follows the element matched by type and key."
  },
  {
    "question": "A ListView with 10,000 items builds slowly. What do you use?",
    "options": [
      "Column",
      "ListView(children: ...)",
      "SingleChildScrollView",
      "ListView.builder so only visible items are built"
    ],
    "correctIndex": 3,
    "explanation": "Lazy lists."
  },
  {
    "question": "What does this print?\n```dart\nString route(String p,bool auth){ if(!auth && p!='/login') return 'redirect'; return 'show $p'; }\nvoid main(){ print(route('/home',false)); print(route('/home',true)); print(route('/login',false)); }\n```\n",
    "options": [
      "show /home\nshow /home\nredirect",
      "redirect\nredirect\nshow /login",
      "redirect\nshow /home\nshow /login",
      "Error"
    ],
    "correctIndex": 2,
    "explanation": "Route guards redirect unauthenticated users. Verified by running it with the Dart SDK."
  },
  {
    "question": "When do you use context.read vs context.watch with Provider?",
    "options": [
      "watch in callbacks",
      "watch in build to rebuild on change; read in callbacks (no subscription)",
      "Same",
      "Reverse"
    ],
    "correctIndex": 1,
    "explanation": "Avoid watch in event handlers."
  },
  {
    "question": "A screen shows a Future but flickers to loading on every rebuild. Why?",
    "options": [
      "Bad theme",
      "Slow network",
      "Key missing",
      "The Future is created inside build; create it once (initState/provider)"
    ],
    "correctIndex": 3,
    "explanation": "FutureBuilder restarts on a new Future."
  },
  {
    "question": "Which states must async UI handle?",
    "options": [
      "None",
      "Data only",
      "Loading, error, empty and data",
      "Loading only"
    ],
    "correctIndex": 2,
    "explanation": "Complete UX."
  },
  {
    "question": "Where should an auth token be stored?",
    "options": [
      "Secure storage (Keychain/Keystore), not shared_preferences",
      "A file",
      "Code",
      "SharedPreferences"
    ],
    "correctIndex": 0,
    "explanation": "Encrypted at rest."
  },
  {
    "question": "What does `tester.pumpAndSettle()` do in a widget test?",
    "options": [
      "Compiles",
      "Sleeps",
      "Navigates",
      "Pumps frames until animations finish and the UI is idle"
    ],
    "correctIndex": 3,
    "explanation": "Deterministic waiting."
  },
  {
    "question": "A CPU-heavy JSON parse janks the UI. What do you do?",
    "options": [
      "Add keys",
      "Run it in an isolate (compute/Isolate.run)",
      "Use setState",
      "Use async"
    ],
    "correctIndex": 1,
    "explanation": "async isn't parallelism."
  },
  {
    "question": "Where does BLoC put business logic?",
    "options": [
      "In main",
      "In widgets",
      "In build",
      "In Bloc/Cubit classes that map events/actions to states, tested without widgets"
    ],
    "correctIndex": 3,
    "explanation": "Plain Dart tests."
  },
  {
    "question": "Which tool tells you what's causing dropped frames?",
    "options": [
      "DevTools performance view in profile mode",
      "flutter analyze",
      "print",
      "Debug mode"
    ],
    "correctIndex": 0,
    "explanation": "Never profile in debug."
  },
  {
    "question": "How do you build dev/staging/prod variants with different API URLs?",
    "options": [
      "Branches",
      "Flavors with --dart-define (or separate entry points) and no committed secrets",
      "Edit code",
      "Comments"
    ],
    "correctIndex": 1,
    "explanation": "Config per environment."
  },
  {
    "question": "What should CI run on every pull request?",
    "options": [
      "flutter analyze, formatting check and flutter test (plus builds)",
      "Nothing",
      "Only build",
      "Manual"
    ],
    "correctIndex": 0,
    "explanation": "Gate merges."
  },
  {
    "question": "A release crashes for 3% of users. How do you limit exposure next time?",
    "options": [
      "Staged rollouts with crash monitoring and a quick rollback path",
      "All at once",
      "Hotfix only",
      "Hope"
    ],
    "correctIndex": 0,
    "explanation": "Play Console/TestFlight phased releases."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Split a bill exactly

Implement splitBill(billCents, tipPercent, people) working in whole cents: validate the input (ArgumentError), compute the total with tip rounded to a cent, and split it so shares differ by at most one cent and add up exactly to the total.

**Starter code**

```dart
/// Split a bill (in cents) with a tip between people so the shares add up exactly.
List<int> splitBill(int billCents, int tipPercent, int people) {
  // throw ArgumentError('invalid input') for negative values or people < 1;
  // total = bill + tip rounded to a whole cent; split so shares differ by at most 1 cent
  // and add up EXACTLY to the total (the first `remainder` people pay one extra cent)
  return [];
}

String money(int cents) => '\$${(cents / 100).toStringAsFixed(2)}';

void main() {
  final shares = splitBill(10000, 18, 3);
  print(shares.map(money).join(' '));
  print(money(shares.reduce((a, b) => a + b)));
  print(splitBill(2500, 0, 4).map(money).join(' '));
  try {
    splitBill(100, 10, 0);
  } on ArgumentError catch (e) {
    print('error: ${e.message}');
  }
}
```

**Expected output**

```text
$39.34 $39.33 $39.33
$118.00
$6.25 $6.25 $6.25 $6.25
error: invalid input
```

<details><summary>Show solution and explanation</summary>

<pre><code>/// Split a bill (in cents) with a tip between people so the shares add up exactly.
List&lt;int&gt; splitBill(int billCents, int tipPercent, int people) {
  if (billCents &lt; 0 || tipPercent &lt; 0 || people &lt; 1) {
    throw ArgumentError(&#x27;invalid input&#x27;);
  }
  final total = (billCents * (100 + tipPercent) / 100).round();
  final base = total ~/ people;
  final remainder = total % people;
  // the first `remainder` people pay one extra cent so nothing is lost to rounding
  return List.generate(people, (i) =&gt; base + (i &lt; remainder ? 1 : 0));
}

String money(int cents) =&gt; &#x27;\$${(cents / 100).toStringAsFixed(2)}&#x27;;

void main() {
  final shares = splitBill(10000, 18, 3);
  print(shares.map(money).join(&#x27; &#x27;));
  print(money(shares.reduce((a, b) =&gt; a + b)));
  print(splitBill(2500, 0, 4).map(money).join(&#x27; &#x27;));
  try {
    splitBill(100, 10, 0);
  } on ArgumentError catch (e) {
    print(&#x27;error: ${e.message}&#x27;);
  }
}
</code></pre>

<p>Money should be handled as integers (cents); the first `remainder` people pay one extra cent so the shares sum exactly instead of losing or inventing a cent to rounding. Verified by running the solution with the Dart SDK.</p>
</details>

### A path matcher with route guards

Implement match(pattern, path) returning the :param map when every segment matches (decoding percent-escapes), else null. The route() function then applies a login guard and renders the matching page, like a minimal go_router.

**Starter code**

```dart
/// A tiny go_router-style matcher: patterns like /book/:id/reviews/:rid
Map<String, String>? match(String pattern, String path) {
  // return the :params map when path matches the pattern segment by segment, else null
  return null;
}

String route(String path, bool loggedIn) {
  if (!loggedIn && path != '/login') return 'redirect -> /login';
  if (loggedIn && path == '/login') return 'redirect -> /';
  for (final entry in {'/': 'HomePage', '/book/:id': 'BookPage', '/book/:id/reviews/:rid': 'ReviewPage', '/login': 'LoginPage'}.entries) {
    final m = match(entry.key, path);
    if (m != null) return '${entry.value} $m';
  }
  return 'NotFoundPage';
}

void main() {
  for (final c in [
    ['/book/42', true],
    ['/book/42/reviews/7', true],
    ['/book/a%20b', true],
    ['/settings', true],
    ['/book/42', false],
    ['/login', true],
  ]) {
    print('${c[0]} (loggedIn=${c[1]}): ${route(c[0] as String, c[1] as bool)}');
  }
}
```

**Expected output**

```text
/book/42 (loggedIn=true): BookPage {id: 42}
/book/42/reviews/7 (loggedIn=true): ReviewPage {id: 42, rid: 7}
/book/a%20b (loggedIn=true): BookPage {id: a b}
/settings (loggedIn=true): NotFoundPage
/book/42 (loggedIn=false): redirect -> /login
/login (loggedIn=true): redirect -> /
```

<details><summary>Show solution and explanation</summary>

<pre><code>/// A tiny go_router-style matcher: patterns like /book/:id/reviews/:rid
Map&lt;String, String&gt;? match(String pattern, String path) {
  final p = pattern.split(&#x27;/&#x27;).where((s) =&gt; s.isNotEmpty).toList();
  final q = path.split(&#x27;/&#x27;).where((s) =&gt; s.isNotEmpty).toList();
  if (p.length != q.length) return null;
  final params = &lt;String, String&gt;{};
  for (var i = 0; i &lt; p.length; i++) {
    if (p[i].startsWith(&#x27;:&#x27;)) {
      params[p[i].substring(1)] = Uri.decodeComponent(q[i]);
    } else if (p[i] != q[i]) {
      return null;
    }
  }
  return params;
}

String route(String path, bool loggedIn) {
  if (!loggedIn &amp;&amp; path != &#x27;/login&#x27;) return &#x27;redirect -&gt; /login&#x27;;
  if (loggedIn &amp;&amp; path == &#x27;/login&#x27;) return &#x27;redirect -&gt; /&#x27;;
  for (final entry in {&#x27;/&#x27;: &#x27;HomePage&#x27;, &#x27;/book/:id&#x27;: &#x27;BookPage&#x27;, &#x27;/book/:id/reviews/:rid&#x27;: &#x27;ReviewPage&#x27;, &#x27;/login&#x27;: &#x27;LoginPage&#x27;}.entries) {
    final m = match(entry.key, path);
    if (m != null) return &#x27;${entry.value} $m&#x27;;
  }
  return &#x27;NotFoundPage&#x27;;
}

void main() {
  for (final c in [
    [&#x27;/book/42&#x27;, true],
    [&#x27;/book/42/reviews/7&#x27;, true],
    [&#x27;/book/a%20b&#x27;, true],
    [&#x27;/settings&#x27;, true],
    [&#x27;/book/42&#x27;, false],
    [&#x27;/login&#x27;, true],
  ]) {
    print(&#x27;${c[0]} (loggedIn=${c[1]}): ${route(c[0] as String, c[1] as bool)}&#x27;);
  }
}
</code></pre>

<p>Declarative routing is a function from the URL (and auth state) to what is on screen: guards redirect first, then path patterns with :params select the page. Verified by running the solution with the Dart SDK.</p>
</details>
