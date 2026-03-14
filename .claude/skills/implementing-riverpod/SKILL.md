---
name: implementing-riverpod
description: Guides Riverpod 2.x state management with code generation (@riverpod). Use when implementing providers, AsyncNotifier controllers, choosing provider types, handling errors with AsyncValue, or deciding keepAlive vs autoDispose. Covers watch/read/listen, family providers, and testing.
---

# Implementing Riverpod

Patterns and best practices for Riverpod 2.x state management with code generation.

## When to Use This Skill

- Implementing new providers or controllers
- Choosing between provider types
- Debugging state management issues
- Understanding watch vs read vs listen
- Handling async state with AsyncValue
- Deciding keepAlive vs autoDispose
- Testing Riverpod providers

## Code Generation Setup

All providers use `@riverpod` annotation:

```yaml
# pubspec.yaml
dependencies:
  flutter_riverpod: ^2.6.1
  riverpod_annotation: ^2.6.1

dev_dependencies:
  riverpod_generator: ^2.6.3
  build_runner: ^2.4.0
```

```bash
# Generate code after changes
dart run build_runner build --delete-conflicting-outputs
```

Generated files: `*.g.dart` (providers), `*.freezed.dart` (immutable state with Freezed).

## Provider Types Quick Reference

| Type | Annotation | Use Case | Example |
|------|-----------|----------|---------|
| Provider | `@riverpod` on function | Computed/derived values, DI | Repository instances |
| FutureProvider | `@riverpod` on async function | One-time async fetch | API calls, config loading |
| StreamProvider | `@riverpod` on Stream function | Real-time data | WebSocket, DB streams |
| Notifier | `@riverpod` on class (sync build) | Sync state + methods | Filters, counters |
| AsyncNotifier | `@riverpod` on class (async build) | Async state + methods | CRUD controllers |

## Core Patterns

### Simple Provider (Dependency Injection)

```dart
part 'providers.g.dart';

@riverpod
ApiClient apiClient(Ref ref) {
  return ApiClient(baseUrl: 'https://api.example.com');
}

@Riverpod(keepAlive: true)
AuthRepository authRepository(Ref ref) {
  return AuthRepository(ref.watch(apiClientProvider));
}
```

### FutureProvider (Async Data)

```dart
@riverpod
Future<User> currentUser(Ref ref) async {
  final repo = ref.watch(userRepositoryProvider);
  return repo.getCurrentUser();
}

// With parameter (family)
@riverpod
Future<Product> product(Ref ref, String id) async {
  final repo = ref.watch(productRepositoryProvider);
  return repo.getProduct(id);
}
// Usage: ref.watch(productProvider('product-123'))
```

### StreamProvider (Real-time Data)

```dart
@riverpod
Stream<List<Message>> messages(Ref ref, String channelId) {
  final repo = ref.watch(messageRepositoryProvider);
  return repo.watchMessages(channelId);
}
```

### AsyncNotifier Controller (Primary Pattern)

```dart
part 'login_controller.g.dart';

@riverpod
class LoginController extends _$LoginController {
  @override
  FutureOr<bool?> build() => null; // Initial state: no action taken

  Future<bool> submit(String email, String password) async {
    state = const AsyncValue.loading();

    state = await AsyncValue.guard(() async {
      final service = ref.read(authServiceProvider);
      await service.signIn(email, password);
      return true;
    });

    return !state.hasError;
  }
}
```

### Notifier (Synchronous State)

```dart
@riverpod
class FilterNotifier extends _$FilterNotifier {
  @override
  ProductFilter build() => const ProductFilter();

  void setCategory(String? category) {
    state = state.copyWith(category: category);
  }

  void setPriceRange(double min, double max) {
    state = state.copyWith(minPrice: min, maxPrice: max);
  }

  void reset() => state = const ProductFilter();
}
```

## watch vs read vs listen

| Method | Where | Rebuilds Widget? | Use Case |
|--------|-------|-----------------|----------|
| `ref.watch()` | build() | Yes | Reactive data display |
| `ref.read()` | callbacks, methods | No | One-time actions (button taps) |
| `ref.listen()` | build() | No | Side effects (navigation, snackbars) |

```dart
class ProductListScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // WATCH: Rebuilds when products change
    final productsAsync = ref.watch(productsProvider);

    // LISTEN: Side effects, no rebuild
    ref.listen(cartProvider, (prev, next) {
      if (next.items.length > (prev?.items.length ?? 0)) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Added to cart')),
        );
      }
    });

    return productsAsync.when(
      data: (products) => ListView.builder(
        itemCount: products.length,
        itemBuilder: (context, i) => ProductCard(
          product: products[i],
          // READ: One-time action in callback
          onAddToCart: () => ref.read(cartProvider.notifier).add(products[i]),
        ),
      ),
      loading: () => const Center(child: CircularProgressIndicator()),
      error: (err, stack) => ErrorDisplay(error: err),
    );
  }
}
```

### Common Mistakes

```dart
// BAD: watch in callback — causes rebuild loop
onPressed: () {
  final value = ref.watch(provider); // WRONG
}

// GOOD: read in callback
onPressed: () {
  final value = ref.read(provider); // CORRECT
}

// BAD: read in build — won't update UI
Widget build(context, ref) {
  final data = ref.read(provider); // WRONG — stale data
}

// GOOD: watch in build
Widget build(context, ref) {
  final data = ref.watch(provider); // CORRECT — reactive
}

// BAD: Forgetting .notifier
ref.read(controllerProvider).submit(); // WRONG — reads state value

// GOOD: Access notifier for methods
ref.read(controllerProvider.notifier).submit(); // CORRECT
```

## AsyncValue Handling

### Pattern 1: .when() (Exhaustive)

```dart
asyncValue.when(
  data: (data) => ContentWidget(data: data),
  loading: () => const CircularProgressIndicator(),
  error: (error, stack) => ErrorWidget(message: error.toString()),
)
```

### Pattern 2: Conditional Checks

```dart
if (state.isLoading) return const LoadingIndicator();
if (state.hasError) return ErrorDisplay(error: state.error!);
final data = state.requireValue;
return ContentWidget(data: data);
```

### Pattern 3: AsyncValue.guard() (Try-Catch Replacement)

```dart
// Replaces try-catch, automatically wraps result or error
state = await AsyncValue.guard(() async {
  return await repository.fetchData();
});
// state is now AsyncData(result) or AsyncError(exception)
```

### Handling Loading + Previous Data

```dart
// Show previous data while loading new data
asyncValue.when(
  skipLoadingOnRefresh: true, // Keep showing old data during refresh
  data: (data) => ContentWidget(data: data),
  loading: () => const LoadingIndicator(),
  error: (e, s) => ErrorWidget(error: e),
)

// Or manually check
if (asyncValue.isLoading && asyncValue.hasValue) {
  // Show data with loading overlay
  return Stack(children: [
    ContentWidget(data: asyncValue.value!),
    const LoadingOverlay(),
  ]);
}
```

## keepAlive vs autoDispose

### Decision Matrix

| Scenario | Lifecycle | Annotation |
|----------|-----------|------------|
| Repository / Service | App lifetime | `@Riverpod(keepAlive: true)` |
| Screen controller | Screen lifetime | `@riverpod` (autoDispose default) |
| User session data | Until logout | `@Riverpod(keepAlive: true)` |
| Search results | While viewing | `@riverpod` |
| Form state | While editing | `@riverpod` |
| Cached API response | Until stale | `@riverpod` + `ref.keepAlive()` conditional |

```dart
// keepAlive: Singleton, lives forever
@Riverpod(keepAlive: true)
DatabaseService databaseService(Ref ref) {
  return DatabaseService();
}

// autoDispose (default): Disposed when no widget watches it
@riverpod
Future<List<SearchResult>> searchResults(Ref ref, String query) async {
  final repo = ref.watch(searchRepositoryProvider);
  return repo.search(query);
}

// Conditional keepAlive: Cache with manual control
@riverpod
Future<UserProfile> userProfile(Ref ref, String userId) async {
  final link = ref.keepAlive(); // Prevent auto-dispose

  // Auto-dispose after 5 minutes of no listeners
  Timer(const Duration(minutes: 5), link.close);

  final repo = ref.watch(userRepositoryProvider);
  return repo.getProfile(userId);
}
```

## Family Providers (Parameterized)

```dart
// Single parameter
@riverpod
Future<Product> product(Ref ref, String id) async {
  return ref.watch(productRepoProvider).getById(id);
}
// Usage: ref.watch(productProvider('abc'))

// Multiple parameters — use a record
@riverpod
Future<List<Product>> filteredProducts(
  Ref ref,
  ({String category, double? maxPrice}) filter,
) async {
  return ref.watch(productRepoProvider).filter(
    category: filter.category,
    maxPrice: filter.maxPrice,
  );
}
// Usage: ref.watch(filteredProductsProvider((category: 'shoes', maxPrice: 100)))
```

## Invalidation & Refresh

```dart
// Invalidate: Mark stale, rebuild on next access
ref.invalidate(productsProvider);

// Refresh: Invalidate + immediate rebuild
final newValue = ref.refresh(productsProvider);

// Invalidate family instance
ref.invalidate(productProvider('abc'));

// Self-invalidation inside notifier
ref.invalidateSelf();
```

## Architecture Integration

```
Widget (ConsumerWidget)
    | ref.watch()
Controller (AsyncNotifier, autoDispose)
    | ref.read()
Service (Provider, keepAlive)
    | ref.watch()
Repository (Provider, keepAlive)
    |
Data Source (API, Database)
```

**Rules:**
- Widgets `watch` controllers
- Controllers `read` services (in methods)
- Services `watch` repositories (in build)
- Repositories and Services are `keepAlive` singletons
- Controllers are `autoDispose` (screen-scoped)

## Provider Scoping & Overrides

```dart
// main.dart
void main() {
  runApp(
    ProviderScope(
      overrides: [
        // Override for environment config
        apiBaseUrlProvider.overrideWithValue('https://staging.api.com'),
      ],
      child: const MyApp(),
    ),
  );
}

// Feature-level scoping
ProviderScope(
  overrides: [themeProvider.overrideWithValue(darkTheme)],
  child: SettingsPage(),
)
```

## Testing

### Unit Testing Providers

```dart
void main() {
  test('productsProvider fetches products', () async {
    final container = ProviderContainer(
      overrides: [
        productRepositoryProvider.overrideWithValue(MockProductRepository()),
      ],
    );
    addTearDown(container.dispose);

    // Arrange
    when(() => container.read(productRepositoryProvider).getAll())
        .thenAnswer((_) async => [Product(id: '1', name: 'Test')]);

    // Act
    final products = await container.read(productsProvider.future);

    // Assert
    expect(products, hasLength(1));
    expect(products.first.name, 'Test');
  });
}
```

### Widget Testing with Riverpod

```dart
testWidgets('ProductList shows products', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      overrides: [
        productsProvider.overrideWith((ref) => [
          Product(id: '1', name: 'Widget Test Product'),
        ]),
      ],
      child: const MaterialApp(home: ProductListScreen()),
    ),
  );

  await tester.pumpAndSettle();
  expect(find.text('Widget Test Product'), findsOneWidget);
});
```

### Testing AsyncNotifier

```dart
test('LoginController submits successfully', () async {
  final container = ProviderContainer(
    overrides: [
      authServiceProvider.overrideWithValue(MockAuthService()),
    ],
  );

  when(() => container.read(authServiceProvider).signIn(any(), any()))
      .thenAnswer((_) async {});

  final notifier = container.read(loginControllerProvider.notifier);
  final result = await notifier.submit('test@test.com', 'password');

  expect(result, true);
  expect(container.read(loginControllerProvider).hasError, false);
});
```

## Common Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| `ref.watch()` in callback | Rebuild loop | Use `ref.read()` |
| `ref.read()` in build | Stale data | Use `ref.watch()` |
| Missing `.notifier` | Reads value instead of controller | `ref.read(p.notifier).method()` |
| `keepAlive` on controllers | Memory leak, stale screen state | Use default autoDispose |
| Forgetting code generation | Provider not found at runtime | Run `build_runner build` |
| Watching in notifier constructor | Lifecycle issue | Watch in `build()` method |

## Quick Checklist (New Provider)

- [ ] Using `@riverpod` annotation with `part '*.g.dart'`?
- [ ] Correct lifecycle? (keepAlive for services, autoDispose for controllers)
- [ ] `watch` in build, `read` in callbacks?
- [ ] AsyncValue handled with `.when()` or `.guard()`?
- [ ] Family parameters use records for multiple values?
- [ ] Tests override dependencies via `ProviderContainer`?
- [ ] Code generation run after changes?
