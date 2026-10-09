---
description: "Build a Flutter signup form with TextFormField, controllers, validators, focus and keyboard actions — and test every path through it."
---

# 06 · Text Input, Forms & Validation

Forms are where users and apps negotiate: the user types something half-right, and the app has
to say exactly what's wrong without being annoying about it. Flutter gives you the pieces —
`TextField`/`TextFormField`, `TextEditingController`, `Form`, validators, `FocusNode` — and leaves
the policy to you. This lesson builds a complete signup form and drives it entirely from widget
tests: typing, tapping, keyboard actions, error messages appearing and disappearing.

## The form

```dart title="signup_form.dart"
import 'package:flutter/material.dart';

/// Validators are top-level functions so they can be unit-tested without widgets.
String? validateEmail(String? v) {
  final value = v?.trim() ?? '';
  if (value.isEmpty) return 'Enter your email';
  if (!RegExp(r'^[^@\s]+@[^@\s]+\.[^@\s]+$').hasMatch(value)) return 'That email looks incomplete';
  return null;
}

String? validateAge(String? v) {
  final n = int.tryParse(v?.trim() ?? '');
  if (n == null) return 'Enter a whole number';
  if (n < 13) return 'You must be 13 or older';
  if (n > 120) return 'Check the age';
  return null;
}

class SignupData {
  const SignupData(this.email, this.age);
  final String email;
  final int age;
}

class SignupForm extends StatefulWidget {
  const SignupForm({super.key, required this.onSubmit});
  final ValueChanged<SignupData> onSubmit;

  @override
  State<SignupForm> createState() => _SignupFormState();
}

class _SignupFormState extends State<SignupForm> {
  final _formKey = GlobalKey<FormState>();
  final _email = TextEditingController();
  final _age = TextEditingController();
  final _ageFocus = FocusNode();
  bool _agreed = false;

  @override
  void dispose() {
    _email.dispose();
    _age.dispose();
    _ageFocus.dispose();
    super.dispose();
  }

  void _submit() {
    final ok = _formKey.currentState!.validate();
    if (!ok || !_agreed) {
      if (!_agreed) {
        ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('Please accept the terms')));
      }
      return;
    }
    widget.onSubmit(SignupData(_email.text.trim(), int.parse(_age.text.trim())));
  }

  @override
  Widget build(BuildContext context) {
    return Form(
      key: _formKey,
      autovalidateMode: AutovalidateMode.onUserInteractionIfError,
      child: Column(
        children: [
          TextFormField(
            key: const Key('email'),
            controller: _email,
            decoration: const InputDecoration(labelText: 'Email'),
            keyboardType: TextInputType.emailAddress,
            textInputAction: TextInputAction.next,
            autofillHints: const [AutofillHints.email],
            validator: validateEmail,
            onFieldSubmitted: (_) => _ageFocus.requestFocus(),
          ),
          TextFormField(
            key: const Key('age'),
            controller: _age,
            focusNode: _ageFocus,
            decoration: const InputDecoration(labelText: 'Age'),
            keyboardType: TextInputType.number,
            textInputAction: TextInputAction.done,
            validator: validateAge,
            onFieldSubmitted: (_) => _submit(),
          ),
          CheckboxListTile(
            value: _agreed,
            onChanged: (v) => setState(() => _agreed = v ?? false),
            title: const Text('I accept the terms'),
          ),
          FilledButton(onPressed: _submit, child: const Text('Create account')),
        ],
      ),
    );
  }
}
```

The moving parts:

- **`TextEditingController`** holds a field's text and selection. You read `_email.text` on submit,
  and you can set it to pre-fill or clear. Controllers own resources, so they're created once (as
  fields of the `State`) and **disposed** in `dispose` — the lifecycle rule from
  [lesson 05](05-stateful-widgets.md).
- **`Form` + `GlobalKey<FormState>`** groups fields. `_formKey.currentState!.validate()` runs every
  field's `validator`, shows the messages, and returns whether all passed.
- **Validators** take the current text and return an error message, or `null` for valid. They're
  top-level functions here so they can be unit-tested without building any widgets.
- **`autovalidateMode: AutovalidateMode.onUserInteractionIfError`** — fields aren't nagged while the
  user is first typing, but once a field has shown an error, it re-validates as they edit, so the
  error disappears the moment it's fixed.
- **Keyboard behaviour**: `keyboardType` picks the on-screen keyboard (email layout, number pad),
  `textInputAction` sets the action key's label and behaviour (*next*, *done*), `onFieldSubmitted`
  reacts to it, and a `FocusNode` lets the email field move focus to the age field.
- **`autofillHints`** let the platform's password manager offer the user's email.
- The terms checkbox isn't a form field, so the submit handler checks it separately and shows a
  `SnackBar`.

## Testing every path

```dart title="signup_form_test.dart"
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hello/l1/signup_form.dart';

void main() {
  late List<SignupData> submitted;
  Future<void> pumpForm(WidgetTester tester) async {
    submitted = [];
    await tester.pumpWidget(MaterialApp(
      home: Scaffold(body: SignupForm(onSubmit: submitted.add)),
    ));
  }

  List<String> errorTexts(WidgetTester tester) => tester
      .widgetList<Text>(find.byType(Text))
      .map((t) => t.data ?? '')
      .where((s) => s.startsWith('Enter') || s.contains('must') || s.contains('looks'))
      .toList();

  testWidgets('empty submit shows every error at once', (tester) async {
    await pumpForm(tester);
    await tester.tap(find.text('Create account'));
    await tester.pump();
    print('errors: ${errorTexts(tester)}');
    expect(submitted, isEmpty);
  });

  testWidgets('errors update as the user fixes them', (tester) async {
    await pumpForm(tester);
    await tester.enterText(find.byKey(const Key('email')), 'ada@');
    await tester.enterText(find.byKey(const Key('age')), '9');
    await tester.tap(find.text('Create account'));
    await tester.pump();
    print('first try:  ${errorTexts(tester)}');
    await tester.enterText(find.byKey(const Key('email')), 'ada@example.com');
    await tester.pump();
    print('email fixed: ${errorTexts(tester)}');
  });

  testWidgets('valid input needs the checkbox too', (tester) async {
    await pumpForm(tester);
    await tester.enterText(find.byKey(const Key('email')), '  ada@example.com ');
    await tester.enterText(find.byKey(const Key('age')), '36');
    await tester.tap(find.text('Create account'));
    await tester.pump();
    print('snackbar shown: ${find.text('Please accept the terms').evaluate().isNotEmpty}');
    await tester.tap(find.text('I accept the terms'));
    await tester.pump();
    await tester.tap(find.text('Create account'));
    await tester.pump();
    print('submitted: ${submitted.map((d) => '${d.email}/${d.age}').toList()}');
  });

  testWidgets('keyboard "next" moves focus to age, "done" submits', (tester) async {
    await pumpForm(tester);
    await tester.tap(find.byKey(const Key('email')));
    await tester.enterText(find.byKey(const Key('email')), 'grace@example.com');
    await tester.testTextInput.receiveAction(TextInputAction.next);
    await tester.pump();
    final ageField = tester.widget<EditableText>(find.descendant(of: find.byKey(const Key('age')), matching: find.byType(EditableText)));
    print('age focused after next: ${ageField.focusNode.hasFocus}');
  });

  test('validators are plain functions', () {
    expect(validateEmail(' a@b.co '), isNull);
    expect(validateEmail('a@b'), 'That email looks incomplete');
    expect(validateAge('130'), 'Check the age');
    expect(validateAge('13'), isNull);
  });
}
```

```console
$ flutter test test/l1/signup_form_test.dart
errors: [Enter your email, Enter a whole number]
first try:  [That email looks incomplete, You must be 13 or older]
email fixed: [You must be 13 or older]
snackbar shown: true
submitted: [ada@example.com/36]
age focused after next: true
00:00 +5: All tests passed!
```

What each test demonstrates:

1. **All errors at once.** Submitting an empty form showed both messages. One `validate()` call runs
   every validator — users fix everything in one pass instead of discovering problems one at a time.
2. **Errors clear as the user fixes them.** After a failed submit, correcting the email removed its
   error immediately on the next frame, without another tap; the age error stayed because that field
   was still wrong. That's `onUserInteractionIfError` at work.
3. **Values are cleaned before use.** The user typed `"  ada@example.com "`; the submitted data has
   the trimmed address. The checkbox gate showed the snackbar first and allowed submission once
   ticked.
4. **Keyboard actions are testable.** `tester.testTextInput.receiveAction(TextInputAction.next)`
   simulates pressing the keyboard's *next* key; focus moved to the age field.
5. **Validators are plain functions** and get plain unit tests — the fastest tests you can write.

`tester.enterText(finder, text)` replaces a field's text (and focuses it); `tester.tap` followed by
`tester.pump()` lets the frame run so the error texts are actually built before asserting.

## Writing good validation messages

- **Say what to do**, not just what's wrong: "Enter a whole number" beats "Invalid".
- **Validate the format on the client, the truth on the server.** The client can't know if an email
  is already registered; the server's answer belongs in the form too (show it with the same error
  style, e.g. via a `String? serverError` you pass into `InputDecoration(errorText: …)`).
- **Don't block typing.** Use `inputFormatters` (e.g. `FilteringTextInputFormatter.digitsOnly`) to
  prevent impossible characters, but keep range checks in validators where the user can see why.
- **Trim, but don't silently rewrite** what users typed in visible fields.

## `TextField` versus `TextFormField`

`TextFormField` is a `TextField` wrapped in a `FormField`, which registers with the nearest `Form` and
adds `validator`, `onSaved` and form-wide reset. Use `TextField` for standalone inputs (a search box,
a chat composer) and `TextFormField` inside forms.

## How It Actually Works

`Form` is a `StatefulWidget` whose `FormState` keeps a set of registered `FormFieldState`s. Each
`TextFormField` registers itself when it builds (it finds the form with `Form.maybeOf(context)`, an
`InheritedWidget` lookup). `validate()` iterates the set, calls each field's `validator` with its
current value, stores the returned message in the field's state and marks it for rebuild; the field's
`InputDecoration` then shows `errorText`.

Typing goes through the platform's text input system: the engine forwards edits to the framework's
`TextInputConnection`, the focused `EditableText` applies them to its controller's `TextEditingValue`
(text plus selection plus composing range — the latter matters for languages typed with an IME), and
the controller notifies listeners, which rebuilds the field. In tests, `TestTextInput` stands in for
the platform side, which is how `enterText` and `receiveAction` work without a keyboard. Focus is a
tree of `FocusNode`s parallel to the widget tree; `requestFocus()` moves primary focus, and the
focused `EditableText` opens the connection that summons the on-screen keyboard.

## Common mistakes

- **Creating controllers in `build`** — the text resets on every rebuild and controllers leak.
- **Forgetting to `dispose` controllers and focus nodes.**
- **Validating on every keystroke from the start** (`AutovalidateMode.always`) — errors appear before
  the user has finished typing.
- **Reading values from controllers but trusting them untrimmed or unparsed.**
- **Checking only the happy path in tests.** Error paths are where forms break.

## Exercise

1. Add a password field with `obscureText: true`, a visibility toggle icon, and a validator requiring
   at least 12 characters. Test the toggle.
2. Add `FilteringTextInputFormatter.digitsOnly` to the age field and write a test that `enterText('4a2')`
   results in `42`.
3. Simulate a server-side "email already registered" error and display it under the email field.
   Clear it when the email text changes.
4. Make "done" on the age field submit the form and write a test using `receiveAction(TextInputAction.done)`.
