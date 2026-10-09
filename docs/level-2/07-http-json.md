---
description: "Call REST APIs from Flutter with package:http: typed JSON parsing with Dart 3 patterns, error handling, timeouts, and testing with MockClient."
---

# 07 · HTTP & JSON

Most apps talk to a server. This lesson builds a small, typed API client for a books service using
`package:http` (1.6.0 here), parses JSON into Dart objects with validation, turns every failure into one
exception type the UI can handle, and tests all of it without touching the network. The design questions
(timeouts, error mapping, where parsing happens) matter more than the specific package: `dio` is a popular
alternative with interceptors built in, and the same structure applies. For REST concepts themselves, see the
[REST API course](https://sigilipelli.github.io/rest-api-mastery-path/).

```bash
flutter pub add http
```

!!! note "Platform permissions"
    Android release builds need `<uses-permission android:name="android.permission.INTERNET"/>` in
    `android/app/src/main/AndroidManifest.xml` (debug builds include it). macOS apps need the
    `com.apple.security.network.client` entitlement. Forgetting these produces apps that work in debug and fail
    in release.

## The client

```dart title="api_client.dart"
import 'dart:convert';
import 'package:http/http.dart' as http;

class Book {
  const Book({required this.id, required this.title, required this.author, this.year});
  final int id;
  final String title;
  final String author;
  final int? year;

  /// Validates while parsing: a malformed payload fails here, not deep in the UI.
  factory Book.fromJson(Map<String, dynamic> json) {
    return switch (json) {
      {'id': int id, 'title': String title, 'author': String author} =>
        Book(id: id, title: title, author: author, year: json['year'] as int?),
      _ => throw FormatException('Not a book: $json'),
    };
  }

  Map<String, dynamic> toJson() => {'id': id, 'title': title, 'author': author, if (year != null) 'year': year};

  @override
  String toString() => 'Book($id, $title, $author, ${year ?? '-'})';
}

/// One exception type for callers to handle.
class ApiException implements Exception {
  ApiException(this.message, {this.statusCode});
  final String message;
  final int? statusCode;
  @override
  String toString() => 'ApiException(${statusCode ?? '-'}): $message';
}

class BooksApi {
  BooksApi({http.Client? client, Uri? baseUrl})
      : _client = client ?? http.Client(),
        _base = baseUrl ?? Uri.parse('https://api.example.com/');
  final http.Client _client;
  final Uri _base;

  Future<List<Book>> list({String? query}) async {
    final uri = _base.resolve('books').replace(queryParameters: {if (query != null) 'q': query});
    final res = await _send(() => _client.get(uri, headers: {'Accept': 'application/json'}));
    final body = jsonDecode(res.body);
    if (body is! List) throw ApiException('Expected a JSON array');
    return body.map((e) => Book.fromJson(e as Map<String, dynamic>)).toList();
  }

  Future<Book> create(String title, String author) async {
    final res = await _send(() => _client.post(
          _base.resolve('books'),
          headers: {'Content-Type': 'application/json'},
          body: jsonEncode({'title': title, 'author': author}),
        ));
    return Book.fromJson(jsonDecode(res.body) as Map<String, dynamic>);
  }

  Future<http.Response> _send(Future<http.Response> Function() request) async {
    final http.Response res;
    try {
      res = await request().timeout(const Duration(seconds: 10));
    } on Exception catch (e) {
      throw ApiException('Network error: $e');
    }
    if (res.statusCode < 200 || res.statusCode >= 300) {
      throw ApiException('HTTP ${res.statusCode}', statusCode: res.statusCode);
    }
    return res;
  }

  void close() => _client.close();
}
```

Design decisions worth copying:

- **The `http.Client` is injected.** Production uses a real `http.Client()`; tests pass a `MockClient`. A client
  also reuses connections across requests, which a bare `http.get(...)` call doesn't, and should be `close()`d
  when you're done with it.
- **URLs are built with `Uri`**, never string concatenation. `replace(queryParameters: ...)` encodes values for you
  (the space in `du ne` becomes `+`).
- **`fromJson` validates.** The Dart 3 map pattern `{'id': int id, 'title': String title, ...}` matches only if the
  keys exist *and* have the right types, binding them to variables in one step. Anything else throws a
  `FormatException` right at the boundary instead of a confusing `type 'String' is not a subtype of type 'int'`
  three screens later.
- **One choke point (`_send`)** applies the timeout, maps transport errors and non-2xx statuses to
  `ApiException`, and keeps each endpoint method short.

## Testing without a network

`package:http/testing.dart` ships `MockClient`, which takes a function from request to response:

```dart title="api_client_test.dart"
import 'dart:convert';
import 'package:flutter_test/flutter_test.dart';
import 'package:http/http.dart' as http;
import 'package:http/testing.dart';
import 'package:l2/s/api_client.dart';

void main() {
  test('list: parses, sends the query, sets headers', () async {
    late http.Request seen;
    final api = BooksApi(client: MockClient((req) async {
      seen = req;
      return http.Response(
        jsonEncode([
          {'id': 1, 'title': 'Dune', 'author': 'Frank Herbert', 'year': 1965},
          {'id': 2, 'title': 'Untitled draft', 'author': 'Me'},
        ]),
        200,
        headers: {'content-type': 'application/json'},
      );
    }));
    final books = await api.list(query: 'du ne');
    print('GET ${seen.url}  accept=${seen.headers['Accept']}');
    print(books);
  });

  test('create: sends JSON, parses the created book', () async {
    final api = BooksApi(client: MockClient((req) async {
      final sent = jsonDecode(req.body) as Map<String, dynamic>;
      print('${req.method} ${req.url.path} body=$sent content-type=${req.headers['Content-Type']}');
      return http.Response(jsonEncode({'id': 42, ...sent}), 201);
    }));
    print(await api.create('Kindred', 'Octavia E. Butler'));
  });

  test('errors become ApiException', () async {
    Future<void> attempt(String label, MockClient client) async {
      try {
        await BooksApi(client: client).list();
      } catch (e) {
        print('$label -> $e');
      }
    }

    await attempt('500', MockClient((_) async => http.Response('oops', 500)));
    await attempt('bad shape', MockClient((_) async => http.Response('{"books": []}', 200)));
    await attempt('bad field', MockClient((_) async => http.Response('[{"id": "1", "title": "X", "author": "Y"}]', 200)));
    await attempt('offline', MockClient((_) async => throw http.ClientException('Connection refused')));
  });
}
```

```console
$ flutter test test/s/api_client_test.dart
GET https://api.example.com/books?q=du+ne  accept=application/json
[Book(1, Dune, Frank Herbert, 1965), Book(2, Untitled draft, Me, -)]
POST /books body={title: Kindred, author: Octavia E. Butler} content-type=application/json
Book(42, Kindred, Octavia E. Butler, -)
500 -> ApiException(500): HTTP 500
bad shape -> ApiException(-): Expected a JSON array
bad field -> FormatException: Not a book: {id: 1, title: X, author: Y}
offline -> ApiException(-): Network error: ClientException: Connection refused
00:00 +3: All tests passed!
```

`api.example.com` is a reserved example domain; nothing was sent over the network. Every test asserts on what the
client *sent* (method, URL encoding, headers, JSON body) as well as what it did with the response. Look at
`bad field`: the server sent `"id": "1"` (a string), the pattern didn't match, and parsing failed loudly. But it
surfaced as a `FormatException`, not an `ApiException` — a gap in our error mapping that exercise 1 asks you to
close.

## Using it from the UI

The client knows nothing about widgets. Expose it through your state layer — for example with Riverpod from
[lesson 05](05-riverpod.md):

```dart
final booksApiProvider = Provider<BooksApi>((ref) {
  final api = BooksApi(baseUrl: Uri.parse(const String.fromEnvironment('API_URL', defaultValue: 'https://api.example.com/')));
  ref.onDispose(api.close);
  return api;
});

final booksProvider = FutureProvider.autoDispose.family<List<Book>, String?>(
  (ref, query) => ref.watch(booksApiProvider).list(query: query),
);
```

and render `AsyncValue` states as in lesson 05, showing `ApiException.message` (or a friendlier mapping of it) in the
error state. `String.fromEnvironment` reads a `--dart-define` value at compile time — [Level 4 · 04](../level-4/04-flavors-config.md)
covers configuration properly.

## Large payloads

`jsonDecode` runs on the UI isolate. For a few kilobytes that's irrelevant; for multi-megabyte responses, decoding can
take long enough to drop frames. `compute(jsonDecode, body)` or `Isolate.run(() => jsonDecode(body))` moves it to a
background isolate — measured and explained in [Level 3 · 06](../level-3/06-isolates.md). Don't do this
pre-emptively; profile first.

## Code generation for models

Hand-written `fromJson` is fine for a handful of models. With dozens, teams typically use `json_serializable` (generates
`fromJson`/`toJson` from annotations) or `freezed` (adds immutable classes, `copyWith`, equality and unions). Both run
through `build_runner`. They save typing, but you still need to decide what happens with missing or mistyped fields —
the generated code throws, so the boundary-validation idea above still applies.

## How It Actually Works

On mobile and desktop, `http.Client()` creates an `IOClient` backed by `dart:io`'s `HttpClient`, which opens TCP
connections (with TLS via the platform's trust store for HTTPS) and keeps them alive for reuse. On the web, the same
`http.Client()` call creates a `BrowserClient` that uses the browser's `fetch`/XHR, so CORS rules apply and some
headers can't be set. (Packages like `cupertino_http` and `cronet_http` provide `Client` implementations backed by
the iOS and Android native networking stacks, which respect system proxies and VPN settings; because they implement
the same interface, swapping them in doesn't change your API code.)

`jsonDecode` produces only `Map<String, dynamic>`, `List<dynamic>`, `String`, `num` (`int` or `double`), `bool` and
`null`. Every type you want beyond that is a conversion you must write — which is why parsing at the boundary into
real types is worth the effort.

## Common mistakes

- **No timeout.** A request with no timeout can hang for a very long time on a bad connection. Always bound it.
- **Treating any response as success.** `http` doesn't throw on 404 or 500; check `statusCode`.
- **`as int` everywhere** in parsing, producing type errors far from the cause. Validate once.
- **Concatenating query strings** (`'$base/books?q=$query'`) — breaks on spaces, `&` and non-ASCII text.
- **Calling `http.get` directly in a widget**, making it untestable and re-requesting on rebuilds (see
  [lesson 06](06-async-ui.md)).
- **Secrets in the app.** Anything compiled into the app can be extracted. API keys that must stay secret belong on a
  server you control.

## Exercise

1. Map `FormatException` (and any other parsing error) to `ApiException` in `list` and `create`, and update the test.
2. Add `Future<void> delete(int id)` that treats `204 No Content` as success and `404` as an `ApiException` whose
   message says the book didn't exist.
3. Add one automatic retry for `ApiException`s with status 502/503/504, with a short delay. Test it with a `MockClient`
   that fails once, then succeeds — and test that a 400 is *not* retried.
4. Write a widget that lists books from `booksProvider`, with a search field, and test it by overriding
   `booksApiProvider` with a `BooksApi` that uses a `MockClient`.
