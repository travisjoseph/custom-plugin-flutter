---
name: testing-quality-assurance
description: Master Flutter testing - unit tests, widget tests, integration tests, mocking with Mockito, CI/CD integration, performance testing, accessibility testing and comprehensive quality assurance strategies.
---

# Testing & Quality Assurance

## Quick Start - Testing Pyramid

```
        /\
       /  \  E2E (10%)
      /────\
     /      \  Integration (20%)
    /────────\
   /          \  Unit (70%)
  /────────────\
```

## Unit Testing

```dart
import 'package:test/test.dart';

// Simple test
void main() {
  group('Calculator', () {
    test('add returns sum', () {
      final calc = Calculator();
      expect(calc.add(2, 3), equals(5));
    });

    test('subtract returns difference', () {
      final calc = Calculator();
      expect(calc.subtract(5, 2), equals(3));
    });
  });
}

// Test exceptions
test('divide by zero throws exception', () {
  final calc = Calculator();
  expect(() => calc.divide(5, 0), throwsException);
});

// Parameterized tests
void main() {
  group('Math', () {
    final testCases = [
      (2, 3, 5),
      (10, 5, 15),
      (1, 1, 2),
    ];

    for (final (a, b, expected) in testCases) {
      test('add($a, $b) == $expected', () {
        expect(Calculator().add(a, b), equals(expected));
      });
    }
  });
}
```

## Widget Testing

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter/material.dart';

void main() {
  testWidgets('Counter increments', (WidgetTester tester) async {
    // Build app
    await tester.pumpWidget(
      const MaterialApp(
        home: CounterApp(),
      ),
    );

    // Verify initial state
    expect(find.text('0'), findsOneWidget);

    // Tap button
    await tester.tap(find.byIcon(Icons.add));
    await tester.pump();  // Rebuild after state change

    // Verify new state
    expect(find.text('1'), findsOneWidget);
  });

  testWidgets('User input in TextField', (tester) async {
    await tester.pumpWidget(
      MaterialApp(
        home: Scaffold(
          body: TextField(
            key: const Key('email_field'),
          ),
        ),
      ),
    );

    // Enter text
    await tester.enterText(
      find.byKey(const Key('email_field')),
      'test@example.com',
    );

    expect(
      find.byKey(const Key('email_field')),
      findsWidgetWithText(TextField, 'test@example.com'),
    );
  });
}

// Advanced finder patterns
final byIcon = find.byIcon(Icons.add);
final byType = find.byType(Button);
final byKey = find.byKey(const Key('my_widget'));
final byText = find.text('Click me');
final bySemantic = find.bySemanticsLabel('Submit');

// Finding with conditions
final specific = find.byWidgetPredicate(
  (widget) => widget is Text && widget.data?.contains('Hello') ?? false,
);

// Ancestor/Descendant finders
final descendant = find.descendant(
  of: find.byType(Column),
  matching: find.byType(Text),
);
```

## Mocking with Mockito

```dart
import 'package:mockito/mockito.dart';

class MockUserRepository extends Mock implements UserRepository {}

void main() {
  group('User Service', () {
    late UserService userService;
    late MockUserRepository mockRepository;

    setUp(() {
      mockRepository = MockUserRepository();
      userService = UserService(mockRepository);
    });

    test('getUser returns user from repository', () async {
      // Arrange
      final user = User(id: '1', name: 'John', email: 'john@example.com');
      when(mockRepository.getUser('1')).thenAnswer((_) async => user);

      // Act
      final result = await userService.getUser('1');

      // Assert
      expect(result, equals(user));
      verify(mockRepository.getUser('1')).called(1);
    });

    test('getUser handles error', () async {
      // Arrange
      when(mockRepository.getUser('1'))
          .thenThrow(Exception('Network error'));

      // Act & Assert
      expect(
        () => userService.getUser('1'),
        throwsException,
      );
    });
  });
}
```

## Integration Testing

```dart
import 'package:flutter_test/flutter_test.dart';

void main() {
  testWidgets('Complete user flow', (tester) async {
    // Launch app
    await tester.pumpWidget(const MyApp());

    // Navigate to login
    await tester.tap(find.text('Login'));
    await tester.pumpAndSettle();

    // Fill form
    await tester.enterText(
      find.byKey(const Key('email')),
      'test@example.com',
    );
    await tester.enterText(
      find.byKey(const Key('password')),
      'password123',
    );

    // Submit
    await tester.tap(find.byType(ElevatedButton));
    await tester.pumpAndSettle();  // Wait for navigation

    // Verify logged in
    expect(find.text('Welcome'), findsOneWidget);
  });
}
```

## BLoC Testing

```dart
import 'package:bloc_test/bloc_test.dart';

void main() {
  group('UserBloc', () {
    late UserBloc userBloc;
    late MockUserRepository mockUserRepository;

    setUp(() {
      mockUserRepository = MockUserRepository();
      userBloc = UserBloc(mockUserRepository);
    });

    blocTest<UserBloc, UserState>(
      'emits [Loading, Success] when getUser succeeds',
      build: () {
        when(mockUserRepository.getUser('1'))
            .thenAnswer((_) async => User(id: '1', name: 'John'));
        return userBloc;
      },
      act: (bloc) => bloc.add(const GetUserEvent('1')),
      expect: () => [
        isA<UserLoading>(),
        isA<UserSuccess>(),
      ],
    );
  });
}
```

## Test Coverage

```bash
# Generate coverage report
flutter test --coverage

# View coverage
lcov --list coverage/lcov.info

# Set minimum coverage threshold
dart test --coverage=coverage
dart tool/check_coverage.dart --minimum=80
```

## CI/CD Testing Integration

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.16.0'

      - name: Get dependencies
        run: flutter pub get

      - name: Run tests
        run: flutter test --coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info
```

## Performance Testing

```dart
testWidgets('Scroll performance', (tester) async {
  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        body: ListView.builder(
          itemCount: 1000,
          itemBuilder: (context, index) {
            return ListTile(title: Text('Item $index'));
          },
        ),
      ),
    ),
  );

  // Measure scroll performance
  await tester.flingFrom(Offset(0, 400), Offset(0, -400), 1000);
  expect(
    tester.binding.window.physicalSize.width,
    greaterThan(0),
  );
});
```

## Accessibility Testing

```dart
testWidgets('Semantic labels present', (tester) async {
  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        floatingActionButton: FloatingActionButton(
          onPressed: () {},
          tooltip: 'Add item',  // ✅ Good
          child: const Icon(Icons.add),
        ),
      ),
    ),
  );

  expect(
    find.bySemanticsLabel('Add item'),
    findsOneWidget,
  );
});
```

## Test Checklist

```
Unit Tests
□ Business logic covered
□ Edge cases tested
□ Error paths handled
□ Dependencies mocked
□ >80% coverage

Widget Tests
□ UI renders correctly
□ User interactions work
□ State changes reflect in UI
□ Navigation works
□ Animations complete

Integration Tests
□ Multi-layer workflows
□ Data persistence
□ API integration
□ Navigation flows

E2E Tests
□ Critical user paths
□ Cross-platform verified
□ Real device tested
□ Performance acceptable

Accessibility
□ Semantic labels present
□ Keyboard navigation works
□ Color contrast valid
□ Touch targets adequate
```

## Best Practices

1. **Test behavior, not implementation**
2. **Use arrange-act-assert pattern**
3. **Mock external dependencies**
4. **Test happy path and error cases**
5. **Aim for 80%+ coverage on business logic**
6. **Use descriptive test names (given-when-then)**
7. **Run tests before committing**
8. **Automate testing in CI/CD**
9. **Test on real devices**
10. **Update tests with code changes**

---

**Build confident, well-tested Flutter apps!**
