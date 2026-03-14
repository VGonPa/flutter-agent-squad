---
name: implementing-bloc
description: Guides BLoC pattern implementation with Dart 3 sealed classes for events and states, Cubit for simpler cases, and event-driven architecture. Use when implementing BLoC/Cubit state management, designing state machines, or testing with bloc_test.
---

# Implementing BLoC

Patterns and best practices for the BLoC (Business Logic Component) pattern with Dart 3 sealed classes and event-driven architecture.

## When to Use This Skill

- Implementing BLoC or Cubit state management
- Designing events and states with sealed classes
- Choosing between Cubit and Bloc
- Testing BLoCs with bloc_test
- Handling complex state transitions
- Migrating from other state management solutions

## Setup

```yaml
# pubspec.yaml
dependencies:
  flutter_bloc: ^9.0.0
  bloc: ^9.0.0
  equatable: ^2.0.5   # Optional: value equality for states

dev_dependencies:
  bloc_test: ^9.0.0
  mocktail: ^1.0.4
```

## Cubit vs Bloc Decision Matrix

| Criteria | Use Cubit | Use Bloc |
|----------|-----------|----------|
| Simple state changes (toggle, counter) | Yes | Overkill |
| No event tracing needed | Yes | - |
| Complex async workflows | - | Yes |
| Need event transformers (debounce, throttle) | - | Yes |
| Multiple events trigger same state | - | Yes |
| Audit trail / event logging required | - | Yes |
| Team prefers explicit event classes | - | Yes |
| Prototype / MVP | Yes | - |

**Rule of thumb:** Start with Cubit. Upgrade to Bloc when you need event transformers, event tracing, or complex multi-step flows.

## States with Dart 3 Sealed Classes

```dart
// auth_state.dart
sealed class AuthState {}

class AuthInitial extends AuthState {}

class AuthLoading extends AuthState {}

class Authenticated extends AuthState {
  final User user;
  Authenticated({required this.user});
}

class Unauthenticated extends AuthState {}

class AuthFailure extends AuthState {
  final String message;
  AuthFailure({required this.message});
}
```

**Why sealed classes:**
- Compiler enforces exhaustive `switch` — no missing cases
- No `is` checks needed in pattern matching
- Subtypes restricted to same file — state set is closed

```dart
// Exhaustive pattern matching — compiler error if case missing
Widget buildForState(AuthState state) => switch (state) {
  AuthInitial() => const SplashScreen(),
  AuthLoading() => const LoadingIndicator(),
  Authenticated(:final user) => HomePage(user: user),
  Unauthenticated() => const LoginPage(),
  AuthFailure(:final message) => ErrorPage(message: message),
};
```

## Events with Sealed Classes

```dart
// auth_event.dart
sealed class AuthEvent {}

class LoginRequested extends AuthEvent {
  final String email;
  final String password;
  LoginRequested({required this.email, required this.password});
}

class LogoutRequested extends AuthEvent {}

class AuthStatusChecked extends AuthEvent {}
```

## Bloc Implementation

```dart
// auth_bloc.dart
class AuthBloc extends Bloc<AuthEvent, AuthState> {
  final LoginUseCase _loginUseCase;
  final LogoutUseCase _logoutUseCase;

  AuthBloc({
    required LoginUseCase loginUseCase,
    required LogoutUseCase logoutUseCase,
  })  : _loginUseCase = loginUseCase,
        _logoutUseCase = logoutUseCase,
        super(AuthInitial()) {
    on<LoginRequested>(_onLoginRequested);
    on<LogoutRequested>(_onLogoutRequested);
    on<AuthStatusChecked>(_onAuthStatusChecked);
  }

  Future<void> _onLoginRequested(
    LoginRequested event,
    Emitter<AuthState> emit,
  ) async {
    emit(AuthLoading());
    final result = await _loginUseCase(event.email, event.password);
    result.fold(
      (failure) => emit(AuthFailure(message: _mapFailure(failure))),
      (user) => emit(Authenticated(user: user)),
    );
  }

  Future<void> _onLogoutRequested(
    LogoutRequested event,
    Emitter<AuthState> emit,
  ) async {
    emit(AuthLoading());
    await _logoutUseCase();
    emit(Unauthenticated());
  }

  Future<void> _onAuthStatusChecked(
    AuthStatusChecked event,
    Emitter<AuthState> emit,
  ) async {
    // Check stored session...
  }

  String _mapFailure(Failure failure) => switch (failure) {
    ServerFailure(:final message) => message ?? 'Server error',
    NetworkFailure() => 'No internet connection',
    _ => 'Something went wrong',
  };
}
```

## Cubit Implementation

```dart
// counter_cubit.dart
class CounterCubit extends Cubit<int> {
  CounterCubit() : super(0);

  void increment() => emit(state + 1);
  void decrement() => emit(state - 1);
  void reset() => emit(0);
}

// More complex cubit with sealed state
class ThemeCubit extends Cubit<ThemeState> {
  final ThemeRepository _repository;

  ThemeCubit(this._repository) : super(ThemeInitial());

  Future<void> loadTheme() async {
    emit(ThemeLoading());
    try {
      final mode = await _repository.getSavedThemeMode();
      emit(ThemeLoaded(mode: mode));
    } catch (e) {
      emit(ThemeLoaded(mode: ThemeMode.system)); // Fallback
    }
  }

  Future<void> setThemeMode(ThemeMode mode) async {
    await _repository.saveThemeMode(mode);
    emit(ThemeLoaded(mode: mode));
  }
}
```

## UI Integration

### BlocProvider (Dependency Injection)

```dart
// Single bloc
BlocProvider(
  create: (context) => AuthBloc(
    loginUseCase: getIt<LoginUseCase>(),
    logoutUseCase: getIt<LogoutUseCase>(),
  ),
  child: const LoginPage(),
)

// Multiple blocs
MultiBlocProvider(
  providers: [
    BlocProvider(create: (_) => getIt<AuthBloc>()),
    BlocProvider(create: (_) => getIt<ThemeCubit>()..loadTheme()),
  ],
  child: const MyApp(),
)
```

### BlocBuilder (Rebuild UI)

```dart
BlocBuilder<AuthBloc, AuthState>(
  builder: (context, state) => switch (state) {
    AuthInitial() => const SplashScreen(),
    AuthLoading() => const Center(child: CircularProgressIndicator()),
    Authenticated(:final user) => HomePage(user: user),
    Unauthenticated() => const LoginPage(),
    AuthFailure(:final message) => ErrorPage(message: message),
  },
)
```

### BlocListener (Side Effects)

```dart
BlocListener<AuthBloc, AuthState>(
  listener: (context, state) => switch (state) {
    Authenticated() => context.go('/home'),
    AuthFailure(:final message) => ScaffoldMessenger.of(context)
        .showSnackBar(SnackBar(content: Text(message))),
    _ => null,
  },
  child: const LoginForm(),
)
```

### BlocConsumer (Build + Listen)

```dart
BlocConsumer<AuthBloc, AuthState>(
  listener: (context, state) {
    if (state is Authenticated) context.go('/home');
    if (state is AuthFailure) {
      ScaffoldMessenger.of(context)
          .showSnackBar(SnackBar(content: Text(state.message)));
    }
  },
  builder: (context, state) {
    if (state is AuthLoading) {
      return const Center(child: CircularProgressIndicator());
    }
    return const LoginForm();
  },
)
```

### Selective Rebuilds with buildWhen

```dart
BlocBuilder<CartBloc, CartState>(
  buildWhen: (previous, current) {
    // Only rebuild when item count changes, not total price
    return previous.itemCount != current.itemCount;
  },
  builder: (context, state) => Badge(
    label: Text('${state.itemCount}'),
    child: const Icon(Icons.shopping_cart),
  ),
)
```

## Event Transformers

```dart
class SearchBloc extends Bloc<SearchEvent, SearchState> {
  SearchBloc(this._repository) : super(SearchInitial()) {
    // Debounce: Wait 300ms after last keystroke
    on<SearchQueryChanged>(
      _onQueryChanged,
      transformer: debounce(const Duration(milliseconds: 300)),
    );

    // Throttle: Max one event per 500ms
    on<SearchScrolledToBottom>(
      _onLoadMore,
      transformer: throttle(const Duration(milliseconds: 500)),
    );
  }
}

// Transformer helpers
EventTransformer<T> debounce<T>(Duration duration) {
  return (events, mapper) => events.debounceTime(duration).flatMap(mapper);
}

EventTransformer<T> throttle<T>(Duration duration) {
  return (events, mapper) => events.throttleTime(duration).flatMap(mapper);
}
```

**Requires:** `package:rxdart` for `debounceTime` and `throttleTime`, or use `bloc_concurrency` package:

```dart
// With bloc_concurrency package (no rxdart needed)
import 'package:bloc_concurrency/bloc_concurrency.dart';

on<SearchQueryChanged>(_onQueryChanged, transformer: restartable());
on<LoadDataRequested>(_onLoadData, transformer: droppable());
on<SaveRequested>(_onSave, transformer: sequential());
```

## Testing with bloc_test

### Testing a Bloc

```dart
void main() {
  late AuthBloc authBloc;
  late MockLoginUseCase mockLogin;
  late MockLogoutUseCase mockLogout;

  setUp(() {
    mockLogin = MockLoginUseCase();
    mockLogout = MockLogoutUseCase();
    authBloc = AuthBloc(loginUseCase: mockLogin, logoutUseCase: mockLogout);
  });

  tearDown(() => authBloc.close());

  blocTest<AuthBloc, AuthState>(
    'emits [AuthLoading, Authenticated] on successful login',
    build: () {
      when(() => mockLogin(any(), any()))
          .thenAnswer((_) async => Right(testUser));
      return authBloc;
    },
    act: (bloc) => bloc.add(
      LoginRequested(email: 'test@test.com', password: 'pass123'),
    ),
    expect: () => [
      isA<AuthLoading>(),
      isA<Authenticated>().having((s) => s.user, 'user', testUser),
    ],
  );

  blocTest<AuthBloc, AuthState>(
    'emits [AuthLoading, AuthFailure] on login failure',
    build: () {
      when(() => mockLogin(any(), any()))
          .thenAnswer((_) async => Left(ServerFailure('Invalid credentials')));
      return authBloc;
    },
    act: (bloc) => bloc.add(
      LoginRequested(email: 'test@test.com', password: 'wrong'),
    ),
    expect: () => [
      isA<AuthLoading>(),
      isA<AuthFailure>().having((s) => s.message, 'message', contains('Invalid')),
    ],
  );
}
```

### Testing a Cubit

```dart
blocTest<CounterCubit, int>(
  'emits [1] when increment is called',
  build: () => CounterCubit(),
  act: (cubit) => cubit.increment(),
  expect: () => [1],
);

blocTest<CounterCubit, int>(
  'emits [1, 2, 3] when increment is called 3 times',
  build: () => CounterCubit(),
  act: (cubit) {
    cubit.increment();
    cubit.increment();
    cubit.increment();
  },
  expect: () => [1, 2, 3],
);
```

### Testing Initial State

```dart
test('initial state is AuthInitial', () {
  expect(authBloc.state, isA<AuthInitial>());
});
```

## Bloc-to-Bloc Communication

```dart
// PREFERRED: Use BlocListener in the widget tree
BlocListener<AuthBloc, AuthState>(
  listener: (context, state) {
    if (state is Unauthenticated) {
      context.read<CartBloc>().add(CartCleared());
    }
  },
  child: ...,
)

// ALTERNATIVE: Stream subscription in Bloc (use sparingly)
class CartBloc extends Bloc<CartEvent, CartState> {
  late final StreamSubscription _authSub;

  CartBloc(AuthBloc authBloc) : super(CartInitial()) {
    _authSub = authBloc.stream.listen((authState) {
      if (authState is Unauthenticated) add(CartCleared());
    });
  }

  @override
  Future<void> close() {
    _authSub.cancel();
    return super.close();
  }
}
```

## Common Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Business logic in widgets | Mixing concerns | Move to Bloc/Cubit |
| Emitting state after close | Runtime error | Check `isClosed` or use `emit` in handler |
| Mutable state classes | Unpredictable behavior | Use sealed classes or Equatable |
| Creating Bloc inside build | New instance every rebuild | Use BlocProvider above widget |
| Bloc-in-Bloc via constructor | Tight coupling | Use BlocListener or stream subscription |
| Missing `buildWhen` on expensive widgets | Unnecessary rebuilds | Add `buildWhen` to filter state changes |

## Quick Checklist (New Bloc/Cubit)

- [ ] States defined as sealed classes?
- [ ] Events defined as sealed classes (Bloc only)?
- [ ] Single responsibility (one feature per Bloc)?
- [ ] Constructor takes use cases/repositories, not raw services?
- [ ] All state transitions tested with `blocTest`?
- [ ] `buildWhen` / `listenWhen` used where needed?
- [ ] Bloc closed properly (BlocProvider handles this)?
- [ ] Event transformers for search/scroll events?
