---
description: "Flutter lists measured: what ListView.builder and ListView(children) really build for thousands of rows, and why stateful rows need keys."
---

# 07 · Lists, Scrolling & Keys

Most app screens are lists: messages, products, settings, search results. Flutter's lists are
lazy — they only build what's on screen — and that laziness, plus the way Flutter matches old
and new widgets, produces the two things to understand in this lesson: how to keep long lists
cheap, and why a list of *stateful* rows needs **keys**. Both are shown with counters and a real
bug rather than rules of thumb.

## Two lists and a stateful row

```dart title="lists.dart"
import 'package:flutter/material.dart';

int rowsBuilt = 0; // how many row build() calls ran
int widgetsCreated = 0; // how many row widget objects were constructed

/// A lazily built list of [count] rows.
class LazyList extends StatelessWidget {
  const LazyList({super.key, required this.count});
  final int count;

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: count,
      itemExtent: 56, // every row is 56px tall: lets the list skip measuring
      itemBuilder: (context, i) {
        widgetsCreated++;
        rowsBuilt++;
        return ListTile(title: Text('Row $i'));
      },
    );
  }
}

/// The same list, built eagerly.
class EagerList extends StatelessWidget {
  const EagerList({super.key, required this.count});
  final int count;

  @override
  Widget build(BuildContext context) {
    return ListView(
      children: [
        for (var i = 0; i < count; i++)
          _counted(Builder(builder: (context) {
            rowsBuilt++;
            return ListTile(title: Text('Row $i'));
          })),
      ],
    );
  }
}

Widget _counted(Widget w) {
  widgetsCreated++;
  return w;
}

/// A row with its own state: a checkbox the user ticks.
class TodoRow extends StatefulWidget {
  const TodoRow({super.key, required this.title});
  final String title;
  @override
  State<TodoRow> createState() => _TodoRowState();
}

class _TodoRowState extends State<TodoRow> {
  bool done = false;
  @override
  Widget build(BuildContext context) => CheckboxListTile(
        value: done,
        onChanged: (v) => setState(() => done = v!),
        title: Text(widget.title),
      );
}

class TodoList extends StatelessWidget {
  const TodoList({super.key, required this.titles, required this.useKeys});
  final List<String> titles;
  final bool useKeys;

  @override
  Widget build(BuildContext context) => ListView(
        children: [
          for (final t in titles) TodoRow(key: useKeys ? ValueKey(t) : null, title: t),
        ],
      );
}
```

- `LazyList` uses `ListView.builder`: it gives Flutter a function to build row *i* on demand.
  `itemExtent: 56` tells the list every row is 56 pixels tall, so it can compute positions without
  laying out rows it skips.
- `EagerList` uses `ListView(children: [...])`, constructing every row widget up front.
- `TodoRow` keeps a `done` flag in its `State`; `TodoList` optionally gives each row a `ValueKey`.

## How much gets built?

```dart title="lists_test.dart"
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hello/l1/lists.dart';

Widget app(Widget body) => MaterialApp(home: Scaffold(body: body));

void main() {
  testWidgets('lazy vs eager building', (tester) async {
    rowsBuilt = widgetsCreated = 0;
    await tester.pumpWidget(app(const LazyList(count: 10000)));
    print('ListView.builder, 10,000 rows: created $widgetsCreated widgets, built $rowsBuilt');

    rowsBuilt = 0;
    await tester.drag(find.byType(ListView), const Offset(0, -3000));
    await tester.pump();
    print('after scrolling 3000px: built $rowsBuilt more; '
        'Row 0 still alive: ${find.text('Row 0').evaluate().isNotEmpty}; '
        'visible now: ${find.text('Row 60').evaluate().isNotEmpty}');

    rowsBuilt = widgetsCreated = 0;
    await tester.pumpWidget(app(const EagerList(count: 2000)));
    print('ListView(children), 2,000 rows: created $widgetsCreated widgets, built $rowsBuilt');
  });

  for (final useKeys in [false, true]) {
    testWidgets('remove the first todo (keys: $useKeys)', (tester) async {
      await tester.pumpWidget(app(TodoList(titles: const ['Buy milk', 'Walk dog', 'Write code'], useKeys: useKeys)));
      await tester.tap(find.text('Walk dog')); // tick "Walk dog"
      await tester.pump();

      await tester.pumpWidget(app(TodoList(titles: const ['Walk dog', 'Write code'], useKeys: useKeys)));
      final ticked = tester
          .widgetList<CheckboxListTile>(find.byType(CheckboxListTile))
          .where((c) => c.value == true)
          .map((c) => (c.title as Text).data)
          .toList();
      print('keys=$useKeys  ticked after removing "Buy milk": $ticked');
    });
  }
}
```

```console
$ flutter test test/l1/lists_test.dart
ListView.builder, 10,000 rows: created 16 widgets, built 16
after scrolling 3000px: built 20 more; Row 0 still alive: false; visible now: true
ListView(children), 2,000 rows: created 2000 widgets, built 16
keys=false  ticked after removing "Buy milk": [Write code]
keys=true  ticked after removing "Buy milk": [Walk dog]
00:00 +3: All tests passed!
```

**`ListView.builder` with 10,000 rows created and built 16.** The 800 × 600 test screen minus the
app's layout fits about eleven 56-pixel rows; the rest are a small **cache extent** — rows just
beyond the edge, built in advance so scrolling doesn't stutter.

**After scrolling 3,000 pixels, 20 more rows were built and `Row 0` was gone.** Rows scrolled out of
view (plus the cache area) are disposed; rows scrolling in are built. Only 20 were built rather than
the ~54 the list scrolled past because the test jumped there in one drag, with no frames in between —
a real fling would build rows for each intermediate frame, but never more than roughly a screenful plus
cache at once. Memory stays flat no matter how long the list is.

**`ListView(children:)` with 2,000 rows created 2,000 widgets but built only 16.** That surprises
people: the children list is still laid out lazily — only visible rows are *built* and get elements
and render objects. The cost is constructing every widget object (and running whatever code produces
the data for each) up front, on every rebuild of the parent. For a handful of rows that's fine and the
syntax is nicer; for long or unknown-length lists, use `.builder` (or `.separated`).

## Keys: the ticked-todo bug

The keys test ticks "Walk dog", then rebuilds the list without "Buy milk" (as if the user deleted
it):

- **Without keys, "Write code" ends up ticked** — the wrong row.
- **With `ValueKey(title)`, "Walk dog" stays ticked** — correct.

Without keys, Flutter matches old and new children **by position and type**. Before: `[TodoRow, TodoRow,
TodoRow]` with the second state ticked. After: `[TodoRow, TodoRow]`. Position 0 keeps state 0 (unticked)
but now shows "Walk dog"; position 1 keeps state 1 (ticked!) but now shows "Write code"; state 2 is
disposed. The *widgets* moved; the *state* didn't.

With keys, Flutter matches by key first: the element whose key is `ValueKey('Walk dog')` is found and
moved to its new position, state and all.

### Choosing a key

| Key | Use when |
|---|---|
| `ValueKey(id)` | items have a stable unique value — a database id is ideal |
| `ObjectKey(obj)` | identity of the object itself is the identity of the row |
| `UniqueKey()` | force a *new* state every build (rarely what you want in lists) |
| `GlobalKey` | access a widget's state from elsewhere, or move a subtree across parents — expensive, use sparingly |

Use a value that identifies the *item*, never the index: `ValueKey(index)` reproduces the bug exactly,
because indexes shift when items are inserted or removed. Titles worked in this test; real apps should
use ids, since titles aren't guaranteed unique.

Stateless rows that hold no state don't strictly need keys, but keys also help Flutter reuse elements
efficiently when rows are reordered, and they're needed for `ReorderableListView` and animated lists.

## Scrolling odds and ends

- `ListView.separated` adds dividers between rows without fake "divider items".
- `GridView.builder` is the two-dimensional equivalent.
- `ScrollController` lets you read or set the scroll offset (e.g. "scroll to top", "load more when
  near the end") — create it in `initState` and dispose it in `dispose`.
- When rows have different heights you can't declare `itemExtent`; Flutter measures rows as they appear,
  and the scrollbar size becomes an estimate.
- Several scrolling sections in one scroll view (a header, a grid, a list) belong in a
  `CustomScrollView` with slivers, not nested `ListView`s ([lesson 04](04-layout-constraints.md)'s
  unbounded-height error is what nesting produces).

## How It Actually Works

A `ListView` is a `ScrollView` that builds a **viewport** containing a **sliver** list. Slivers are
render objects that lay out pieces of a scrollable area: instead of box constraints, a sliver receives
`SliverConstraints` — the current scroll offset, how much space is visible, and the cache extent — and
reports how much of itself it painted. `RenderSliverList` (or `RenderSliverFixedExtentList`, used when
`itemExtent` is set) asks its child manager to create children only for the indexes that intersect the
visible region plus cache, and garbage-collects children that leave it. The child manager is a
`SliverMultiBoxAdaptorElement` that calls your `itemBuilder` — or, for `ListView(children:)`, indexes
into the prebuilt list — which is why both forms build lazily but differ in construction cost.

Key matching happens in `Element.updateChildren`, the algorithm that reconciles an old list of child
elements with a new list of widgets. It walks both lists from the start and the end while widgets can
update elements in place (same type and key), then puts the remaining old children with keys into a map
and looks up each remaining new widget by its key. Unkeyed children in the middle are matched in order —
which is precisely the behaviour that moved the tick.

## Common mistakes

- **`ListView(children: data.map(...).toList())` for long lists** — every row widget constructed on every
  rebuild.
- **`shrinkWrap: true` inside another scroll view** to make an error go away — it lays out every row.
- **Index keys** (`ValueKey(i)`) on lists that can change.
- **Stateful rows without keys** in lists that support delete, insert or reorder.
- **Doing expensive work in `itemBuilder`** (parsing, formatting dates for 10,000 items in advance) —
  prepare data once, build rows cheaply.

## Exercise

1. Add `ListView.separated` with a `Divider` and confirm with the counter that separators are built
   lazily too.
2. Change the keyed test to use `ValueKey(index)` and confirm the bug returns.
3. Add a "Load more" behaviour: when the user scrolls within 200 pixels of the end, append 50 rows.
   Test it with `tester.drag` and `ScrollController.position`.
4. Turn `TodoList` into a `ReorderableListView`, drag a ticked row to the top in a test, and check the
   tick moves with it.
