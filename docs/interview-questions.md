---
title: "Flutter Interview Questions and Answers (179 Questions)"
description: "179 Flutter interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Flutter Mastery Path."
---

# Flutter Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Flutter Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What Flutter Is: Engine, Widgets &amp; One Codebase](level-1/01-what-flutter-is.md)

**Q1.** What is Flutter?

**Answer:** A UI toolkit for building apps for mobile, web and desktop from one Dart codebase

It draws its own pixels.

**Q2.** What distinguishes Flutter from many cross-platform tools?

**Answer:** It renders its own UI with its own engine rather than wrapping native widgets

Consistent look and performance.

**Q3.** What language does Flutter use?

**Answer:** Dart

AOT for release, JIT for hot reload.

**Q4.** What does hot reload do?

**Answer:** Injects updated code into the running app preserving state

Fast iteration.

### [02 · Setup, flutter create &amp; Project Anatomy](level-1/02-setup-first-app.md)

**Q5.** Which command checks your Flutter installation?

**Answer:** flutter doctor

Fix reported issues.

**Q6.** What does `flutter analyze` do?

**Answer:** Statically analyzes code for errors and lint issues

Part of CI.

**Q7.** What does `flutter test` run?

**Answer:** Dart unit and widget tests

integration_test needs a device.

**Q8.** What is the entry point?

**Answer:** main() calling runApp(widget)

Root widget.

### [03 · Widgets, build() &amp; Composition](level-1/03-widgets-composition.md)

**Q9.** What does 'composition' mean in Flutter UI?

**Answer:** Building UI by nesting small widgets rather than inheriting or drawing

Card &gt; Padding &gt; Row.

**Q10.** What does build() return?

**Answer:** A widget tree describing the UI for the current state

Should be cheap and pure.

**Q11.** Why prefer many small widgets?

**Answer:** Reuse, readability and efficient rebuilds

const where possible.

**Q12.** What does `const` on a widget enable?

**Answer:** Reusing identical instances and skipping rebuilds

Cheaper builds.

### [04 · Layout: Constraints Go Down, Sizes Go Up](level-1/04-layout-constraints.md)

**Q13.** Flutter's layout rule is...

**Answer:** Constraints go down, sizes go up, parents set positions

Debug by following it.

**Q14.** What does Expanded do in a Row?

**Answer:** Makes a child take the remaining space

Needs a Flex parent.

**Q15.** Why does 'RenderFlex overflowed' occur?

**Answer:** Children need more space than the parent's constraints allow

Use Expanded/Flexible/scroll.

**Q16.** What does a ConstrainedBox do?

**Answer:** Imposes additional size constraints on its child

Understand constraints.

### [05 · StatefulWidget, setState &amp; the State Lifecycle](level-1/05-stateful-widgets.md)

**Q17.** When do you need a StatefulWidget?

**Answer:** When the UI depends on mutable data that changes over time

Counters, forms, animations.

**Q18.** What does setState do?

**Answer:** Marks the State dirty so build runs again

Only for local state.

**Q19.** What's the lifecycle method to release resources?

**Answer:** dispose()

Dispose controllers.

**Q20.** Where do you start timers/subscriptions?

**Answer:** initState() (and cancel in dispose())

Avoid in build.

### [06 · Text Input, Forms &amp; Validation](level-1/06-forms-input.md)

**Q21.** What does a Form with GlobalKey enable?

**Answer:** Validating all fields together via formKey.currentState!.validate()

TextFormField validators.

**Q22.** What's a validator's return?

**Answer:** An error string or null if valid

null = valid.

**Q23.** Why dispose TextEditingControllers?

**Answer:** Avoid memory leaks

In dispose().

**Q24.** When validate on interaction?

**Answer:** AutovalidateMode.onUserInteraction avoids nagging before input

Good UX.

### [07 · Lists, Scrolling &amp; Keys](level-1/07-lists-keys.md)

**Q25.** Why use ListView.builder?

**Answer:** It builds only visible items lazily, good for long lists

Efficient.

**Q26.** What are Keys for?

**Answer:** Preserving state when list items are reordered or removed

ValueKey/ObjectKey.

**Q27.** What happens without keys when reordering stateful items?

**Answer:** State can attach to the wrong item

Use stable keys.

**Q28.** Why give itemExtent?

**Answer:** Faster layout when items have a fixed size

Optional optimization.

### [08 · Material 3 Theming](level-1/08-theming.md)

**Q29.** Where is app-wide styling defined?

**Answer:** ThemeData (colorScheme, textTheme) in MaterialApp

Centralize.

**Q30.** How do you read the theme in a widget?

**Answer:** Theme.of(context)

Inherited lookup.

**Q31.** What enables dark mode?

**Answer:** darkTheme plus themeMode in MaterialApp

System setting.

**Q32.** Why use ColorScheme.fromSeed?

**Answer:** Generate a coherent Material 3 palette from one color

Consistent.

### [09 · Assets, Images &amp; Fonts](level-1/09-assets.md)

**Q33.** Where are assets declared?

**Answer:** pubspec.yaml under flutter: assets:

Then flutter pub get.

**Q34.** How do you display a bundled image?

**Answer:** Image.asset('assets/logo.png')

Path must match.

**Q35.** Why include resolution variants?

**Answer:** Pick the right density for the device

2.0x/3.0x folders.

**Q36.** What loads a bundled JSON?

**Answer:** rootBundle.loadString('assets/data.json')

Declared asset.

### [10 · Project — A Tip Splitter with Widget Tests](level-1/10-project-tip-splitter.md)

**Q37.** What does this print (tip splitter logic)?

```dart
double split(double bill,int tipPct,int people){ var total=bill*(1+tipPct/100); return total/people; }
void main(){ print(split(120,15,4).toStringAsFixed(2)); }
```

**Answer:** 34.50

Total with tip divided evenly. Verified by running it with the Dart SDK.

**Q38.** What does the project combine?

**Answer:** Composition, layout, state, forms and theming in a tip splitter

Level 1 project.

**Q39.** How do you handle invalid input?

**Answer:** Validate and show clear errors, disabling calculation until valid

UX.

**Q40.** How do you test it?

**Answer:** Widget tests entering values and asserting displayed totals

Automated.

## Level 2 · Intermediate

### [01 · Navigation: push, pop &amp; Returning Results](level-2/01-navigator.md)

**Q41.** What is Navigator's model?

**Answer:** A stack of routes: push adds, pop removes

Declarative alternatives exist.

**Q42.** How do you return data from a pushed route?

**Answer:** Navigator.pop(context, result) and await the push

Futures.

**Q43.** What does pushReplacement do?

**Answer:** Replaces the current route in the stack

Login → home.

**Q44.** Why check `mounted` after an await before using context?

**Answer:** The widget may be disposed meanwhile

Avoid context use across async gaps.

### [02 · Declarative Routing with go_router](level-2/02-go-router.md)

**Q45.** What does go_router add?

**Answer:** URL-based declarative routing, deep links and redirects

Web-friendly.

**Q46.** What does `redirect` do?

**Answer:** Sends users elsewhere based on state (e.g. auth)

Guards.

**Q47.** How do you pass parameters?

**Answer:** Path parameters like /book/:id and query params

state.pathParameters.

**Q48.** Why use named routes/paths constants?

**Answer:** Avoid typos and ease refactoring

Maintainability.

### [03 · Lifting State &amp; InheritedWidget](level-2/03-inherited-widget.md)

**Q49.** What is an InheritedWidget for?

**Answer:** Sharing data down the tree and rebuilding dependents when it changes

of(context).

**Q50.** What triggers dependents to rebuild?

**Answer:** updateShouldNotify returning true

Compare old/new.

**Q51.** What does dependOnInheritedWidgetOfExactType do?

**Answer:** Registers the dependency so the widget rebuilds on change

vs getElementForInheritedWidget.

**Q52.** Why is it the basis of Provider/Theme/MediaQuery?

**Answer:** They build on the same inherited lookup mechanism

Core concept.

### [04 · Provider &amp; ChangeNotifier](level-2/04-provider.md)

**Q53.** What does Provider offer over raw InheritedWidget?

**Answer:** Less boilerplate, lifecycle management and convenient read/watch APIs

ChangeNotifierProvider.

**Q54.** Which call rebuilds on change?

**Answer:** context.watch&lt;T&gt;()

read is for callbacks.

**Q55.** Why use context.read in onPressed?

**Answer:** It doesn't subscribe; you don't want rebuilds from a callback

Avoid watch in handlers.

**Q56.** What does ChangeNotifier require after state changes?

**Answer:** notifyListeners()

Triggers rebuilds.

### [05 · Riverpod](level-2/05-riverpod.md)

**Q57.** What problem does Riverpod fix?

**Answer:** Providers not found due to tree position, global access without context and testability issues

Compile-safe.

**Q58.** What does ref.watch do?

**Answer:** Reads a provider and rebuilds when it changes

Reactive.

**Q59.** What's a FutureProvider for?

**Answer:** Exposing async data with loading/error/data states

AsyncValue.

**Q60.** How do you override a provider in tests?

**Answer:** ProviderScope(overrides: [...])

Isolation.

### [06 · Async UI: FutureBuilder &amp; StreamBuilder](level-2/06-async-ui.md)

**Q61.** What does FutureBuilder show?

**Answer:** Different UI for loading, error and data states of a Future

Handle all states.

**Q62.** Why not create the Future inside build?

**Answer:** It would restart on every rebuild

Create once (initState/provider).

**Q63.** What's StreamBuilder for?

**Answer:** Rebuilding UI as stream events arrive

Live data.

**Q64.** What should async UI always handle?

**Answer:** Loading, error and empty states

Good UX.

### [07 · HTTP &amp; JSON](level-2/07-http-json.md)

**Q65.** How do you parse JSON into typed objects?

**Answer:** Factory constructors fromJson with validation

Fail clearly on bad data.

**Q66.** What HTTP package is used?

**Answer:** package:http (or dio)

Client with timeouts.

**Q67.** How should errors be handled?

**Answer:** Map failures to typed exceptions/results shown in the UI

One error model.

**Q68.** Why set timeouts?

**Answer:** Avoid hanging UIs on slow networks

User feedback.

### [08 · Local Persistence](level-2/08-persistence.md)

**Q69.** When use shared_preferences?

**Answer:** Small key/value settings

Simple persistence.

**Q70.** When choose a database (sqflite/Drift/Isar)?

**Answer:** Structured, queryable or larger datasets

Shape of the data.

**Q71.** Where store secrets/tokens?

**Answer:** Secure storage (Keychain/Keystore via flutter_secure_storage)

Encrypted.

**Q72.** Why persist state?

**Answer:** The OS can kill your app at any time

Restore.

### [09 · Testing: Unit, Widget &amp; Golden Tests](level-2/09-testing.md)

**Q73.** What are the test types?

**Answer:** Unit, widget and integration (plus golden tests)

Pyramid.

**Q74.** What does `tester.pump()` do?

**Answer:** Advances a frame so the UI updates

pumpAndSettle waits for animations.

**Q75.** What are golden tests?

**Answer:** Compare rendered widgets to stored reference images

Catch visual regressions.

**Q76.** How control dependencies in widget tests?

**Answer:** Inject fakes via providers/constructors

Deterministic.

### [10 · Project — A Reading List App](level-2/10-project-reading-list.md)

**Q77.** What does the reading list project combine?

**Answer:** Navigation, state management, HTTP/JSON, persistence and tests

Level 2 project.

**Q78.** How do you keep layers separate?

**Answer:** UI, state/controllers and repositories

Maintainable.

**Q79.** How do you test the repository?

**Answer:** With a fake HTTP client returning canned JSON

Fast.

**Q80.** How handle offline?

**Answer:** Cache locally and sync when online

Resilience.

## Level 3 · Advanced

### [01 · Widgets, Elements &amp; BuildContext](level-3/01-element-tree.md)

**Q81.** What are the three trees?

**Answer:** Widget, Element and RenderObject trees

Widgets are immutable configs.

**Q82.** What does an Element do?

**Answer:** Holds the instance of a widget in the tree and manages updates

Reuses where types/keys match.

**Q83.** Why are widgets cheap to rebuild?

**Answer:** They're lightweight descriptions; Elements/RenderObjects persist

Diffing.

**Q84.** What decides whether an Element updates or is replaced?

**Answer:** Matching runtimeType and key

Widget.canUpdate.

### [02 · Animations: Implicit &amp; Explicit](level-3/02-animations.md)

**Q85.** What's an implicit animation widget?

**Answer:** AnimatedContainer/AnimatedOpacity that animate property changes automatically

Simple.

**Q86.** What does an AnimationController need?

**Answer:** A TickerProvider (vsync) and disposal

SingleTickerProviderStateMixin.

**Q87.** What does a Tween do?

**Answer:** Maps animation progress 0..1 to a value range

Combine with curves.

**Q88.** Why dispose controllers?

**Answer:** Stop tickers and avoid leaks

dispose().

### [03 · CustomPainter &amp; Drawing](level-3/03-custom-painter.md)

**Q89.** When use CustomPainter?

**Answer:** For custom 2D drawing on a Canvas

paint(Canvas, Size).

**Q90.** What does shouldRepaint control?

**Answer:** Whether the painter needs to repaint when updated

Performance.

**Q91.** Why wrap in RepaintBoundary?

**Answer:** Isolate repaint to reduce work

Profiling.

**Q92.** What coordinate origin does Canvas use?

**Answer:** Top-left with y down

Remember in math.

### [04 · Responsive &amp; Adaptive Layouts](level-3/04-responsive.md)

**Q93.** What builds adaptive layouts?

**Answer:** LayoutBuilder/MediaQuery with breakpoints

Constraints-based.

**Q94.** What's better than MediaQuery for local sizing?

**Answer:** LayoutBuilder (uses parent constraints)

Component-level.

**Q95.** How handle text scaling?

**Answer:** Respect textScaleFactor and avoid fixed heights

Accessibility.

**Q96.** What's a common breakpoint approach?

**Answer:** Compact/medium/expanded width classes

Material guidance.

### [05 · Platform Channels](level-3/05-platform-channels.md)

**Q97.** What are platform channels?

**Answer:** Message passing between Dart and native (Kotlin/Swift) code

MethodChannel.

**Q98.** When use them?

**Answer:** To access platform APIs without a plugin

Prefer existing plugins.

**Q99.** What types cross the channel?

**Answer:** Standard message codec types (num, String, List, Map, bytes)

Serialize.

**Q100.** What must you handle?

**Answer:** PlatformException and missing implementations

Errors.

### [06 · Isolates &amp; Heavy Work](level-3/06-isolates.md)

**Q101.** When use isolates in Flutter?

**Answer:** To run CPU-heavy work off the UI thread (compute/Isolate.run)

Avoid jank.

**Q102.** What do isolates share?

**Answer:** Nothing: they communicate by message passing

Copy semantics.

**Q103.** Is async/await parallel?

**Answer:** No, it's concurrency on one isolate

Use isolates for CPU work.

**Q104.** What symptom indicates UI-thread overload?

**Answer:** Dropped frames/jank in the performance overlay

Profile.

### [07 · The Bloc Pattern](level-3/07-bloc.md)

**Q105.** What is BLoC?

**Answer:** A pattern separating business logic into event→state streams/classes

flutter_bloc.

**Q106.** What does a Bloc consume and emit?

**Answer:** Events in, states out

Predictable flow.

**Q107.** Why is BLoC testable?

**Answer:** Logic is plain Dart tested with events and expected states

bloc_test.

**Q108.** What does BlocBuilder do?

**Answer:** Rebuilds the UI when the Bloc's state changes

Select states.

### [08 · Accessibility &amp; Internationalization](level-3/08-a11y-i18n.md)

**Q109.** What's required for accessibility?

**Answer:** Semantics labels, sufficient contrast and tap target sizes

Screen readers.

**Q110.** How do you localize strings?

**Answer:** ARB files and generated AppLocalizations (intl)

flutter gen-l10n.

**Q111.** Why avoid hard-coded text?

**Answer:** It prevents translation and complicates changes

i18n.

**Q112.** What should you test?

**Answer:** With a screen reader and large text settings

Real usage.

### [09 · Integration Tests &amp; Test Strategy](level-3/09-integration-tests.md)

**Q113.** What do integration tests run?

**Answer:** The full app on a device/emulator end to end

integration_test.

**Q114.** How find widgets?

**Answer:** find.byKey/byType/text with the WidgetTester

Stable keys.

**Q115.** Why keep integration tests few?

**Answer:** They're slower and flakier than unit/widget tests

Pyramid.

**Q116.** What reduces flakiness?

**Answer:** Waiting on conditions (pumpAndSettle) rather than fixed delays

Deterministic.

### [10 · Project — An Offline-First Habit Tracker](level-3/10-project-habit-tracker.md)

**Q117.** What does the habit tracker combine?

**Answer:** State management, persistence, animations, accessibility and tests

Level 3 project.

**Q118.** How do you store habit history?

**Answer:** Local database with date-keyed records

Query streaks.

**Q119.** What does this compute (streak logic)?

```dart
int streak(List<bool> done){ var s=0; for(var i=done.length-1;i>=0&&done[i];i--){s++;} return s; }
void main(){ print(streak([true,false,true,true,true])); print(streak([true,true,false])); }
```

**Answer:** 3
0

Count trailing completed days. Verified by running it with the Dart SDK.

**Q120.** What should widgets show for an empty history?

**Answer:** A friendly empty state

UX.

## Level 4 · Master

### [01 · The Rendering Pipeline &amp; RenderObjects](level-4/01-rendering-pipeline.md)

**Q121.** What are the rendering pipeline phases?

**Answer:** Build, layout, paint and composite

Per frame.

**Q122.** What's the frame budget at 60 fps?

**Answer:** About 16 ms

UI and raster threads.

**Q123.** What causes jank?

**Answer:** Expensive work in build/layout/paint exceeding the frame budget

DevTools.

**Q124.** What does the raster thread do?

**Answer:** Executes GPU commands to draw the frame

Impeller/Skia.

### [02 · Performance: Rebuilds, Jank &amp; DevTools](level-4/02-performance.md)

**Q125.** What's the first step to improve performance?

**Answer:** Profile in profile mode with DevTools

Never in debug.

**Q126.** Which reduces rebuilds?

**Answer:** const widgets, splitting widgets and scoping state

Rebuild only what changes.

**Q127.** What causes expensive layouts?

**Answer:** Deep intrinsic sizing and nested scrollables

Avoid IntrinsicHeight in lists.

**Q128.** How to avoid image jank?

**Answer:** Resize/cache images and use cacheWidth/cacheHeight

Decode costs.

### [03 · App Architecture: Layers &amp; Dependency Injection](level-4/03-architecture.md)

**Q129.** What does clean architecture separate?

**Answer:** UI, domain logic and data layers with dependency rules

Testability.

**Q130.** What's the repository pattern?

**Answer:** An abstraction over data sources for the domain layer

Swap implementations.

**Q131.** Why use dependency injection?

**Answer:** Replace implementations in tests and configure per environment

get_it/Riverpod.

**Q132.** What's a good rule for state management choice?

**Answer:** Be consistent and choose what the team can maintain

Pragmatic.

### [04 · Configuration, Flavors &amp; --dart-define](level-4/04-flavors-config.md)

**Q133.** What are flavors?

**Answer:** Build variants (dev/staging/prod) with different config

--flavor.

**Q134.** How supply config?

**Answer:** --dart-define values or env-specific entry points

No secrets in repo.

**Q135.** What differs per flavor?

**Answer:** App ID, API base URL, icons and feature flags

Environment.

**Q136.** Why not commit secrets?

**Answer:** They leak via source control and binaries are extractable

Use backends.

### [05 · CI for Flutter](level-4/05-ci.md)

**Q137.** What should CI run for Flutter?

**Answer:** flutter analyze, flutter test and builds

Gate merges.

**Q138.** Why cache pub and Flutter SDK in CI?

**Answer:** Faster runs

Pin versions.

**Q139.** What does `dart format --set-exit-if-changed` enforce?

**Answer:** Consistent formatting

CI check.

**Q140.** How to ship builds from CI?

**Answer:** Fastlane/Codemagic with signing secrets in the CI vault

Automated releases.

### [06 · Building &amp; Publishing to the Stores](level-4/06-publishing.md)

**Q141.** What's needed to publish on Google Play?

**Answer:** A signed app bundle (.aab) and a Play developer account

Upload key.

**Q142.** What's needed for the App Store?

**Answer:** Xcode archive, provisioning/signing and an Apple developer account

TestFlight.

**Q143.** Why use staged rollouts?

**Answer:** Limit exposure of a bad release

Monitor crashes.

**Q144.** What does obfuscation (--obfuscate) do?

**Answer:** Makes Dart symbols harder to read (keep symbol files for crash reports)

Debug info.

### [07 · Flutter on the Web &amp; Desktop](level-4/07-web-desktop.md)

**Q145.** What differs on web/desktop?

**Answer:** Input modes, layouts, plugins and performance characteristics

Responsive and adaptive.

**Q146.** What's a web renderer concern?

**Answer:** Bundle size and first load; consider deferred loading

Performance.

**Q147.** How adapt to mouse/keyboard?

**Answer:** Hover, focus, shortcuts and scrollbars

Desktop UX.

**Q148.** What plugin check matters?

**Answer:** Platform support listed for each plugin

pub.dev platforms.

### [08 · Plugins &amp; dart:ffi](level-4/08-plugins-ffi.md)

**Q149.** What does FFI allow?

**Answer:** Calling native C libraries from Dart

dart:ffi.

**Q150.** When write a plugin?

**Answer:** To expose platform functionality reusable across apps

federated structure.

**Q151.** What do you ship in a plugin package?

**Answer:** Dart API, per-platform implementations, example app and tests

Quality.

**Q152.** How test platform code?

**Answer:** Integration tests on devices plus unit tests of Dart wrappers

Cover both sides.

### [09 · Upgrades, Dependencies &amp; Long-Lived Apps](level-4/09-upgrades.md)

**Q153.** What's a safe Flutter upgrade approach?

**Answer:** Read the breaking-changes notes, upgrade on a branch, fix analyzer issues and run tests

flutter upgrade.

**Q154.** What does `dart fix --apply` do?

**Answer:** Applies automated fixes for deprecations

Helpful for migrations.

**Q155.** Why pin the Flutter version in CI?

**Answer:** Reproducible builds

fvm.

**Q156.** How detect dependency issues?

**Answer:** flutter pub outdated and resolving conflicts deliberately

Maintenance.

### [10 · Capstone — A Production-Ready Expense Tracker](level-4/10-capstone.md)

**Q157.** What does the capstone integrate?

**Answer:** Architecture, state, persistence, tests, CI and release configuration

Production app.

**Q158.** How do you prove readiness?

**Answer:** Tests, profiling, accessibility checks and staged rollout

Quality gates.

**Q159.** What documentation helps?

**Answer:** README, architecture decisions and release runbooks

Onboarding.

**Q160.** What sustains the app?

**Answer:** Monitoring crashes and regular dependency/SDK updates

Maintenance.

## Mixed questions across all levels

**Q161.** What makes Flutter different from wrappers around native widgets?

**Answer:** It renders its own UI with its own engine, so look and behavior are consistent across platforms

Everything is a widget drawn by Flutter.

**Q162.** What is the Flutter layout rule that explains most overflow bugs?

**Answer:** Constraints go down, sizes go up, parents set positions

Debug by following it.

**Q163.** A Row with three long Text widgets overflows. What do you do?

**Answer:** Wrap children in Expanded/Flexible (or allow wrapping/scrolling)

Row gives unbounded main-axis constraints.

**Q164.** A counter must change when a button is pressed. What type of widget do you need?

**Answer:** A StatefulWidget (or state from a state-management solution) calling setState

State changes trigger rebuilds.

**Q165.** You start a Timer in a State. Where do you cancel it?

**Answer:** In dispose()

Avoid leaks and setState after dispose.

**Q166.** A reorderable list of stateful tiles mixes up their text field contents. What's missing?

**Answer:** Stable Keys (ValueKey/ObjectKey) on the items

State follows the element matched by type and key.

**Q167.** A ListView with 10,000 items builds slowly. What do you use?

**Answer:** ListView.builder so only visible items are built

Lazy lists.

**Q168.** What does this print?

```dart
String route(String p,bool auth){ if(!auth && p!='/login') return 'redirect'; return 'show $p'; }
void main(){ print(route('/home',false)); print(route('/home',true)); print(route('/login',false)); }
```

**Answer:** redirect
show /home
show /login

Route guards redirect unauthenticated users. Verified by running it with the Dart SDK.

**Q169.** When do you use context.read vs context.watch with Provider?

**Answer:** watch in build to rebuild on change; read in callbacks (no subscription)

Avoid watch in event handlers.

**Q170.** A screen shows a Future but flickers to loading on every rebuild. Why?

**Answer:** The Future is created inside build; create it once (initState/provider)

FutureBuilder restarts on a new Future.

**Q171.** Which states must async UI handle?

**Answer:** Loading, error, empty and data

Complete UX.

**Q172.** Where should an auth token be stored?

**Answer:** Secure storage (Keychain/Keystore), not shared_preferences

Encrypted at rest.

**Q173.** What does `tester.pumpAndSettle()` do in a widget test?

**Answer:** Pumps frames until animations finish and the UI is idle

Deterministic waiting.

**Q174.** A CPU-heavy JSON parse janks the UI. What do you do?

**Answer:** Run it in an isolate (compute/Isolate.run)

async isn't parallelism.

**Q175.** Where does BLoC put business logic?

**Answer:** In Bloc/Cubit classes that map events/actions to states, tested without widgets

Plain Dart tests.

**Q176.** Which tool tells you what's causing dropped frames?

**Answer:** DevTools performance view in profile mode

Never profile in debug.

**Q177.** How do you build dev/staging/prod variants with different API URLs?

**Answer:** Flavors with --dart-define (or separate entry points) and no committed secrets

Config per environment.

**Q178.** What should CI run on every pull request?

**Answer:** flutter analyze, formatting check and flutter test (plus builds)

Gate merges.

**Q179.** A release crashes for 3% of users. How do you limit exposure next time?

**Answer:** Staged rollouts with crash monitoring and a quick rollback path

Play Console/TestFlight phased releases.

## Practical coding questions

### Split a bill exactly

Implement splitBill(billCents, tipPercent, people) working in whole cents: validate the input (ArgumentError), compute the total with tip rounded to a cent, and split it so shares differ by at most one cent and add up exactly to the total.

Starter code:

```
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

Expected output:

```text
$39.34 $39.33 $39.33
$118.00
$6.25 $6.25 $6.25 $6.25
error: invalid input
```

??? success "Show a solution"

    ```
    /// Split a bill (in cents) with a tip between people so the shares add up exactly.
    List<int> splitBill(int billCents, int tipPercent, int people) {
      if (billCents < 0 || tipPercent < 0 || people < 1) {
        throw ArgumentError('invalid input');
      }
      final total = (billCents * (100 + tipPercent) / 100).round();
      final base = total ~/ people;
      final remainder = total % people;
      // the first `remainder` people pay one extra cent so nothing is lost to rounding
      return List.generate(people, (i) => base + (i < remainder ? 1 : 0));
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

    Money should be handled as integers (cents); the first `remainder` people pay one extra cent so the shares sum exactly instead of losing or inventing a cent to rounding. Verified by running the solution with the Dart SDK.

### A path matcher with route guards

Implement match(pattern, path) returning the :param map when every segment matches (decoding percent-escapes), else null. The route() function then applies a login guard and renders the matching page, like a minimal go_router.

Starter code:

```
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

Expected output:

```text
/book/42 (loggedIn=true): BookPage {id: 42}
/book/42/reviews/7 (loggedIn=true): ReviewPage {id: 42, rid: 7}
/book/a%20b (loggedIn=true): BookPage {id: a b}
/settings (loggedIn=true): NotFoundPage
/book/42 (loggedIn=false): redirect -> /login
/login (loggedIn=true): redirect -> /
```

??? success "Show a solution"

    ```
    /// A tiny go_router-style matcher: patterns like /book/:id/reviews/:rid
    Map<String, String>? match(String pattern, String path) {
      final p = pattern.split('/').where((s) => s.isNotEmpty).toList();
      final q = path.split('/').where((s) => s.isNotEmpty).toList();
      if (p.length != q.length) return null;
      final params = <String, String>{};
      for (var i = 0; i < p.length; i++) {
        if (p[i].startsWith(':')) {
          params[p[i].substring(1)] = Uri.decodeComponent(q[i]);
        } else if (p[i] != q[i]) {
          return null;
        }
      }
      return params;
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

    Declarative routing is a function from the URL (and auth state) to what is on screen: guards redirect first, then path patterns with :params select the page. Verified by running the solution with the Dart SDK.

---

More free practice: the [Flutter Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
