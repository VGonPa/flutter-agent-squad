---
name: testing-flutter
description: Guides Flutter testing strategy and implementation. Use when deciding what to test, writing unit/widget/integration/golden tests, mocking with mocktail, testing state management (Riverpod, BLoC), or structuring test suites. Covers Arrange-Act-Assert, coverage targets, and common pitfalls.
---

# Testing Flutter

Testing strategy, patterns, and decision guide for Flutter applications.

## When to Use This Skill

- Deciding what tests to write for new code
- Choosing between unit, widget, integration, or golden tests
- Writing mocks with mocktail or mockito
- Testing Riverpod providers or BLoC state
- Debugging flaky or failing tests
- Structuring test files and naming conventions

## Testing Philosophy

### What to Test (Priority Order)

| Priority | What | Why | Coverage Target |
|----------|------|-----|-----------------|
| 1 | Business logic (Services/Use Cases) | Core value, complex rules | >=90% |
| 2 | State management (Controllers/BLoC) | User-facing behavior | >=80% |
| 3 | Repositories (data access) | Data integrity | >=80% |
| 4 | Widgets (user interactions) | UX correctness | Key flows |
| 5 | Models (if has logic) | Data consistency | Methods only |

### What NOT to Test

- **Generated code** (`.g.dart`, `.freezed.dart`) - Already tested by package authors
- **Framework code** - Flutter/Riverpod/BLoC internals work
- **Trivial getters** - No logic to verify
- **UI styling** - Use golden tests sparingly, they're brittle

## Test Type Decision Guide

```
"I need to test..."

Business logic, calculations, rules?
  -> Unit test (fast, isolated)

State changes in response to actions?
  -> Unit test with ProviderContainer or blocTest

Widget renders correctly for different states?
  -> Widget test with provider/BLoC overrides

User flow across multiple screens?
  -> Integration test (slow, use sparingly)

Visual appearance hasn't changed?
  -> Golden test (brittle, maintain carefully)
```

## Arrange-Act-Assert Pattern

Every test must have three clear sections:

```dart
test('should return user when found', () async {
  // Arrange - Set up mocks, create test data, configure state
  when(() => mockRepository.getUser('123'))
      .thenAnswer((_) async => testUser);

  // Act - Execute the single action being tested
  final result = await service.getUser('123');

  // Assert - Verify the expected outcome
  expect(result, equals(testUser));
});
```

**Why:** Tests become documentation. Anyone reading knows exactly what's being tested.

## Unit Tests

### Testing Services / Use Cases

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

class MockUserRepository extends Mock implements UserRepository {}

void main() {
  late UserService service;
  late MockUserRepository mockRepository;

  setUp(() {
    mockRepository = MockUserRepository();
    service = UserService(mockRepository);
  });

  group('UserService', () {
    group('getUser', () {
      test('should return user when repository finds one', () async {
        // Arrange
        when(() => mockRepository.getUser('123'))
            .thenAnswer((_) async => testUser);

        // Act
        final result = await service.getUser('123');

        // Assert
        expect(result, equals(testUser));
        verify(() => mockRepository.getUser('123')).called(1);
      });

      test('should throw when user not found', () async {
        // Arrange
        when(() => mockRepository.getUser('999'))
            .thenThrow(UserNotFoundException());

        // Act & Assert
        expect(
          () => service.getUser('999'),
          throwsA(isA<UserNotFoundException>()),
        );
      });
    });
  });
}
```

### Testing Models

```dart
group('Product', () {
  test('fromJson creates Product correctly', () {
    final json = {'id': '1', 'name': 'Test', 'price': 9.99};
    final product = Product.fromJson(json);

    expect(product.id, '1');
    expect(product.name, 'Test');
    expect(product.price, 9.99);
  });

  test('copyWith creates new instance with updated values', () {
    final product = Product(id: '1', name: 'Test', price: 9.99);
    final updated = product.copyWith(price: 19.99);

    expect(updated.price, 19.99);
    expect(product.price, 9.99); // Original unchanged
  });
});
```

## Widget Tests

### Basic Widget Test

```dart
testWidgets('ProductCard displays product info', (tester) async {
  // Arrange
  final product = Product(id: '1', name: 'Widget', price: 9.99);

  // Act
  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(body: ProductCard(product: product)),
    ),
  );

  // Assert
  expect(find.text('Widget'), findsOneWidget);
  expect(find.text('\$9.99'), findsOneWidget);
});

testWidgets('ProductCard tap triggers callback', (tester) async {
  bool tapped = false;

  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        body: ProductCard(
          product: testProduct,
          onTap: () => tapped = true,
        ),
      ),
    ),
  );

  await tester.tap(find.byType(ProductCard));
  await tester.pump();

  expect(tapped, true);
});
```

### Widget Test Timing

| Method | Use When |
|--------|----------|
| `pump()` | Advance one frame, control timing precisely |
| `pump(duration)` | Wait specific time (animations, debounce) |
| `pumpAndSettle()` | Wait until no more frames scheduled |

**Pitfall:** `pumpAndSettle()` times out if there's an infinite animation (loading spinner). Use `pump()` instead.

### Test Helper for Pumping Widgets

```dart
// test/helpers/pump_app.dart
extension WidgetTesterX on WidgetTester {
  Future<void> pumpApp(Widget widget) async {
    await pumpWidget(
      MaterialApp(home: widget),
    );
  }
}
```

## State Management Testing

### Riverpod with ProviderContainer

```dart
test('counterProvider increments correctly', () {
  final container = ProviderContainer();
  addTearDown(container.dispose);

  expect(container.read(counterProvider), 0);
  container.read(counterProvider.notifier).increment();
  expect(container.read(counterProvider), 1);
});

test('authProvider returns authenticated state', () async {
  final mockRepository = MockAuthRepository();
  when(() => mockRepository.getCurrentUser())
      .thenAnswer((_) async => testUser);

  final container = ProviderContainer(
    overrides: [
      authRepositoryProvider.overrideWithValue(mockRepository),
    ],
  );
  addTearDown(container.dispose);

  final authState = await container.read(authProvider.future);
  expect(authState, isA<Authenticated>());
});
```

**Why `addTearDown(container.dispose)`:** Riverpod containers hold state. Without disposal: memory leaks, state bleeds across tests, flaky failures.

### Riverpod Widget Testing

```dart
testWidgets('displays user name from provider', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      overrides: [
        userProvider.overrideWith((ref) => testUser),
      ],
      child: const MaterialApp(home: UserPage()),
    ),
  );

  expect(find.text(testUser.name), findsOneWidget);
});
```

### BLoC Testing with blocTest

```dart
import 'package:bloc_test/bloc_test.dart';

void main() {
  late ProductsBloc bloc;
  late MockGetProducts mockGetProducts;

  setUp(() {
    mockGetProducts = MockGetProducts();
    bloc = ProductsBloc(getProducts: mockGetProducts);
  });

  tearDown(() => bloc.close());

  test('initial state is ProductsInitial', () {
    expect(bloc.state, ProductsInitial());
  });

  blocTest<ProductsBloc, ProductsState>(
    'emits [Loading, Loaded] when LoadProducts succeeds',
    build: () {
      when(() => mockGetProducts())
          .thenAnswer((_) async => testProducts);
      return bloc;
    },
    act: (bloc) => bloc.add(LoadProducts()),
    expect: () => [
      ProductsLoading(),
      ProductsLoaded(testProducts),
    ],
  );

  blocTest<ProductsBloc, ProductsState>(
    'emits [Loading, Error] when LoadProducts fails',
    build: () {
      when(() => mockGetProducts())
          .thenThrow(ServerException('Failed'));
      return bloc;
    },
    act: (bloc) => bloc.add(LoadProducts()),
    expect: () => [
      ProductsLoading(),
      ProductsError('Failed to load products'),
    ],
  );
}
```

## Mocking

### Mocktail (Preferred - No Code Generation)

```dart
import 'package:mocktail/mocktail.dart';

// Create mocks
class MockUserRepository extends Mock implements UserRepository {}

// Create fakes for any() matchers
class FakeUser extends Fake implements User {}

void main() {
  setUpAll(() {
    registerFallbackValue(FakeUser());
  });

  // Use in tests
  when(() => mockRepo.save(any())).thenAnswer((_) async => true);
  verify(() => mockRepo.save(any())).called(1);
}
```

### Fakes vs Mocks

| Type | Use When | Example |
|------|----------|---------|
| Mock | Verify calls, stub returns | `MockUserRepository` |
| Fake | Need as parameter to `any()` | `FakeUser` |

### Mock at Repository Level

```
Controller (real)
    |
Service (real)         <-- Tests real business logic
    |
Repository (MOCKED)   <-- Mock here
    |
Firebase (never touched)
```

## Integration Tests

```dart
// integration_test/app_test.dart
import 'package:integration_test/integration_test.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('login and browse flow', (tester) async {
    app.main();
    await tester.pumpAndSettle();

    await tester.enterText(find.byKey(Key('email_field')), 'test@example.com');
    await tester.enterText(find.byKey(Key('password_field')), 'password123');
    await tester.tap(find.byKey(Key('login_button')));
    await tester.pumpAndSettle();

    expect(find.text('Welcome'), findsOneWidget);
  });
}
```

## Golden Tests

```dart
testWidgets('ProductCard matches golden', (tester) async {
  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(body: ProductCard(product: testProduct)),
    ),
  );

  await expectLater(
    find.byType(ProductCard),
    matchesGoldenFile('goldens/product_card.png'),
  );
});
```

```bash
# Update golden files
flutter test --update-goldens
```

## Test Organization

### File Structure

```
test/
├── unit/
│   ├── models/
│   ├── repositories/
│   ├── services/
│   └── utils/
├── widget/
│   ├── pages/
│   └── widgets/
├── integration/
├── mocks/           # Shared mock classes
├── fixtures/        # JSON test data
└── helpers/         # pumpApp, test utilities
```

### Test Naming

```dart
group('ClassName', () {
  group('methodName', () {
    test('should [expected behavior] when [condition]', () {});
  });
});
```

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Testing implementation, not behavior | Test "should return user" not "should call repo.get()" |
| Too many assertions per test | One test = one behavior |
| Shared mutable state between tests | Each test sets up its own mocks in `setUp()` |
| Not testing error paths | Happy path is obvious; errors are where bugs hide |
| Flaky async tests | Missing `await`, wrong pump timing, or state leaking |
| `pumpAndSettle` timeout | Infinite animation present; use `pump()` instead |

## Coverage

```bash
flutter test --coverage
genhtml coverage/lcov.info -o coverage/html
open coverage/html/index.html
```

| Component | Target | Notes |
|-----------|--------|-------|
| Services | >=90% | All business logic paths |
| Controllers/BLoC | >=80% | All state transitions |
| Repositories | >=80% | CRUD + error handling |
| Widgets | Key flows | Don't chase 100% |

**100% coverage is NOT the goal.** Meaningful tests that catch bugs are.
