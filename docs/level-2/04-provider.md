---
description: "Share app state with Provider and ChangeNotifier: watch vs read vs select, measured rebuild counts, disposal, and injecting state into widget tests."
---

# 04 · Provider & ChangeNotifier

[Lesson 03](03-inherited-widget.md) built an `InheritedWidget` by hand: a stateful host, a scope, an `of`
method. `provider` is a package that turns that pattern into a one-liner, adds lifecycle management
(it creates and disposes your objects), and gives you three precise ways to read state. It has been one of the
most widely used Flutter state approaches for years and is still a sound default for small and medium apps. This
lesson used provider 6.1.5+1.

```bash
flutter pub add provider
```

## The model: a ChangeNotifier

`ChangeNotifier` (part of Flutter's foundation library) is a minimal observable: `addListener`,
`removeListener`, and `notifyListeners()`. Put your state and the operations on it in a subclass:

```dart title="provider_demo.dart (model)"
class Cart extends ChangeNotifier {
  final Map<String, int> _qty = {};

  int quantityOf(String id) => _qty[id] ?? 0;
  int get itemCount => _qty.values.fold(0, (a, b) => a + b);
  int get totalCents => catalog.fold(0, (sum, p) => sum + p.priceCents * quantityOf(p.id));

  void add(Product p) {
    _qty.update(p.id, (q) => q + 1, ifAbsent: () => 1);
    notifyListeners();
  }

  void remove(Product p) { ... notifyListeners(); }
}
```

Note what's *not* here: no `BuildContext`, no widgets. The cart can be unit-tested with plain `test()`, and the
UI is a thin layer that displays it and calls its methods. Prices are integer cents, for the same reason as in
the [tip splitter project](../level-1/10-project-tip-splitter.md).

## Providing and consuming

The full file:

```dart title="provider_demo.dart"
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

class Product {
  const Product(this.id, this.name, this.priceCents);
  final String id;
  final String name;
  final int priceCents;
}

const catalog = [
  Product('p1', 'Notebook', 450),
  Product('p2', 'Pen', 120),
  Product('p3', 'Backpack', 3999),
];

/// The model: plain Dart, testable without Flutter widgets.
class Cart extends ChangeNotifier {
  final Map<String, int> _qty = {};

  int quantityOf(String id) => _qty[id] ?? 0;
  int get itemCount => _qty.values.fold(0, (a, b) => a + b);
  int get totalCents => catalog.fold(0, (sum, p) => sum + p.priceCents * quantityOf(p.id));

  void add(Product p) {
    _qty.update(p.id, (q) => q + 1, ifAbsent: () => 1);
    notifyListeners();
  }

  void remove(Product p) {
    final q = quantityOf(p.id);
    if (q <= 1) {
      _qty.remove(p.id);
    } else {
      _qty[p.id] = q - 1;
    }
    notifyListeners();
  }
}

final builds = <String>[];

String money(int cents) => '\$${(cents ~/ 100)}.${(cents % 100).toString().padLeft(2, '0')}';

class ShopPage extends StatelessWidget {
  const ShopPage({super.key});
  @override
  Widget build(BuildContext context) {
    builds.add('ShopPage');
    return Scaffold(
      appBar: AppBar(title: const Text('Shop'), actions: const [CartBadge()]),
      body: ListView(children: [for (final p in catalog) ProductRow(product: p)]),
      bottomNavigationBar: const TotalBar(),
    );
  }
}

class CartBadge extends StatelessWidget {
  const CartBadge({super.key});
  @override
  Widget build(BuildContext context) {
    // select: rebuild only when itemCount changes.
    final count = context.select<Cart, int>((c) => c.itemCount);
    builds.add('CartBadge');
    return Padding(padding: const EdgeInsets.all(16), child: Text('Cart: $count', key: const Key('badge')));
  }
}

class ProductRow extends StatelessWidget {
  const ProductRow({super.key, required this.product});
  final Product product;
  @override
  Widget build(BuildContext context) {
    final qty = context.select<Cart, int>((c) => c.quantityOf(product.id));
    builds.add('Row ${product.name}');
    // read: get the object for calling methods, without subscribing.
    final cart = context.read<Cart>();
    return ListTile(
      title: Text(product.name),
      subtitle: Text('${money(product.priceCents)} × $qty'),
      trailing: Row(mainAxisSize: MainAxisSize.min, children: [
        IconButton(key: Key('remove-${product.id}'), icon: const Icon(Icons.remove), onPressed: qty == 0 ? null : () => cart.remove(product)),
        IconButton(key: Key('add-${product.id}'), icon: const Icon(Icons.add), onPressed: () => cart.add(product)),
      ]),
    );
  }
}

class TotalBar extends StatelessWidget {
  const TotalBar({super.key});
  @override
  Widget build(BuildContext context) {
    // watch: rebuild on every notifyListeners().
    final cart = context.watch<Cart>();
    builds.add('TotalBar');
    return Padding(padding: const EdgeInsets.all(16), child: Text('Total ${money(cart.totalCents)}', key: const Key('total')));
  }
}

Widget buildShopApp({Cart? cart}) => ChangeNotifierProvider(
      create: (_) => cart ?? Cart(),
      child: const MaterialApp(home: ShopPage()),
    );
```

`ChangeNotifierProvider(create: ...)` sits above `MaterialApp`, so every route can reach the cart. Below it,
there are three ways to get at it, and choosing correctly is most of the skill:

| Call | Subscribes? | Use it in | For |
|---|---|---|---|
| `context.watch<Cart>()` | yes, to every notification | `build` | widgets that display lots of the model |
| `context.select<Cart, T>((c) => ...)` | yes, but rebuilds only if the selected value changes (`==`) | `build` | widgets that need one derived value |
| `context.read<Cart>()` | no | callbacks (`onPressed`) or `build` when you only need methods | calling methods |

`Consumer<Cart>(builder: (context, cart, child) => ...)` and `Selector<Cart, T>` are the widget forms of `watch`
and `select`; they're useful when you want to subscribe a small part of a big `build` method without extracting a
new widget.

## Measuring rebuilds

```dart title="provider_test.dart"
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:l2/s/provider_demo.dart';

void main() {
  test('Cart is plain Dart', () {
    final cart = Cart();
    var notifications = 0;
    cart.addListener(() => notifications++);
    cart..add(catalog[0])..add(catalog[0])..add(catalog[1])..remove(catalog[0]);
    print('items=${cart.itemCount} total=${money(cart.totalCents)} notifications=$notifications');
  });

  testWidgets('select limits rebuilds', (tester) async {
    await tester.pumpWidget(buildShopApp());
    builds.clear();

    await tester.tap(find.byKey(const Key('add-p2')));
    await tester.pump();
    print('add pen   -> rebuilt $builds');
    print('           ${tester.widget<Text>(find.byKey(const Key('badge'))).data}, ${tester.widget<Text>(find.byKey(const Key('total'))).data}');

    builds.clear();
    await tester.tap(find.byKey(const Key('add-p2')));
    await tester.pump();
    print('add pen   -> rebuilt $builds');

    builds.clear();
    await tester.tap(find.byKey(const Key('add-p3')));
    await tester.pump();
    print('add bag   -> rebuilt $builds');
    print('           ${tester.widget<Text>(find.byKey(const Key('total'))).data}');
  });

  testWidgets('tests can inject a pre-filled cart', (tester) async {
    final cart = Cart()..add(catalog[2])..add(catalog[2]);
    await tester.pumpWidget(buildShopApp(cart: cart));
    print('injected: ${tester.widget<Text>(find.byKey(const Key('total'))).data}');
  });
}
```

```console
$ flutter test test/s/provider_test.dart
items=2 total=$5.70 notifications=4
add pen   -> rebuilt [TotalBar, CartBadge, Row Pen]
           Cart: 1, Total $1.20
add pen   -> rebuilt [TotalBar, CartBadge, Row Pen]
add bag   -> rebuilt [TotalBar, CartBadge, Row Backpack]
           Total $42.39
injected: Total $79.98
00:00 +3: All tests passed!
```

- **The model test** ran with no widgets at all: four mutations, four notifications, correct totals.
- **Adding a pen** rebuilt three widgets — the total (it `watch`es), the badge (its `itemCount` changed) and the
  pen's own row (its quantity changed). `ShopPage`, the Notebook row and the Backpack row didn't rebuild: their
  selected values didn't change. With `watch` everywhere, every row would rebuild on every tap; with 3 products
  nobody would notice, with 300 you might.
- **Injection** is a constructor argument away. Because `buildShopApp` accepts a `Cart`, a test can set up any
  state it needs before the first frame.

## Lifecycle: `create` versus `.value`

`ChangeNotifierProvider(create: (_) => Cart())` **owns** the cart: it creates it lazily the first time someone
reads it and calls `dispose()` on it when the provider leaves the tree. `ChangeNotifierProvider.value(value:
existingCart)` only *exposes* an object someone else owns and never disposes it. Use `create` for new objects and
`.value` when passing an existing object down (for example into a new route), never the other way round:
`.value(value: Cart())` inside a `build` method creates a new cart on every rebuild.

## Combining providers

Real apps have several: `MultiProvider(providers: [...], child: ...)` flattens the nesting. When one object needs
another — an `OrderService` that needs the `ApiClient` — `ProxyProvider` builds it from its dependencies and
rebuilds it when they change. If you find yourself writing `ProxyProvider3`, it's a sign to consider a DI
approach ([Level 4 · 03](../level-4/03-architecture.md)) or Riverpod, which handles dependencies between
providers natively.

## How It Actually Works

`provider` is a layer over `InheritedWidget`. `ChangeNotifierProvider` creates a private inherited element that
holds your object, and subscribes to it with `addListener`. When the cart calls `notifyListeners()`, the provider
marks its inherited element as changed and notifies dependents — the same `didChangeDependencies` path you saw in
lesson 03.

`context.watch` is `dependOnInheritedWidgetOfExactType` on that element. `context.read` uses
`getElementForInheritedWidgetOfExactType` — a lookup without registering a dependency, which is why calling it in
a callback is safe and calling `watch` in a callback is an error. `context.select` registers a dependency *with a
filter*: the provider's element remembers the selector and its last result for each dependent, and on
notification re-runs the selector and only marks the dependent dirty if the result changed. That's why selectors
must be cheap and return values with meaningful `==` (an `int`, a `String`, an immutable object) — returning a new
`List` each time defeats it.

## Common mistakes

- **`watch` in `onPressed`.** Throws an assertion: you can't subscribe from outside `build`. Use `read`.
- **`read` in `build` to display data.** The widget never updates. If it shows the value, it should `watch` or
  `select` it.
- **Forgetting `notifyListeners()`** after a mutation. The model changes; nothing on screen does.
- **Mutating a list returned by a getter** from the outside (`cart.items.add(...)`) — bypasses
  `notifyListeners()`. Expose unmodifiable views.
- **`ProviderNotFoundException`** because the provider is below `MaterialApp` on one route and you pushed a new
  route. Providers must be *above* the `Navigator` to be visible on all routes.

## Exercise

1. Add a `CartPage` reached from the badge that lists only items with quantity > 0 and has a "Clear cart" button.
   Make the badge reachable from both pages without moving the provider.
2. Replace `TotalBar`'s `watch` with `select`. Does its rebuild count change? Why or why not?
3. Write a `select` that returns `catalog.where(...).toList()` and count rebuilds. Explain the result, then fix it.
4. Persist the cart with the tools from [lesson 08](08-persistence.md): which object should own the save logic?
