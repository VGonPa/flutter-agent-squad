---
name: designing-flutter-architecture
description: Guides Clean Architecture implementation for Flutter apps. Use when deciding which layer code belongs in, structuring new features, or establishing dependency rules. Covers model/data/domain/presentation layers, feature-based organization, and file structure conventions.
---

# Designing Flutter Architecture

Guide for implementing Clean Architecture in Flutter applications with feature-based organization.

## When to Use This Skill

- Starting a new Flutter project and defining structure
- Deciding which layer new code belongs in
- Understanding dependency flow between layers
- Structuring a new feature module
- Refactoring code to the correct layer
- Resolving architecture violations

## Layer Overview

```
┌─────────────────────────────────────────────┐
│              PRESENTATION                    │
│  ┌─────────────┐    ┌─────────────────────┐ │
│  │   Widgets   │ <> │    Controllers      │ │
│  │    (UI)     │    │  (Riverpod)         │ │
│  └─────────────┘    └─────────────────────┘ │
├─────────────────────────────────────────────┤
│                DOMAIN                        │
│  ┌──────────────────┐  ┌─────────────────┐ │
│  │   Use Cases       │  │   Entities      │ │
│  │   Repo Interfaces │  │   Value Objects │ │
│  └──────────────────┘  └─────────────────┘ │
├─────────────────────────────────────────────┤
│                 DATA                         │
│  ┌─────────────────────────────────────────┐│
│  │   Repositories (impl), Data Sources     ││
│  │   DTOs, Mappers                         ││
│  └─────────────────────────────────────────┘│
└─────────────────────────────────────────────┘
```

## Dependency Rule

**Dependencies flow inward only:**

```
Presentation → Domain ← Data
```

- Presentation depends on Domain
- Data depends on Domain (implements its interfaces)
- Domain depends on **nothing** (pure Dart, no Flutter imports)

**NEVER:**
- Domain imports Presentation or Data
- Presentation imports Data directly (use Domain interfaces)
- Data imports Presentation

## Quick Decision Guide

| I need to...                  | Layer        | Example                           |
|-------------------------------|--------------|-----------------------------------|
| Show UI                       | Presentation | `ProductCard`, `LoginScreen`      |
| Manage screen state           | Presentation | `LoginController`, `CartController` |
| Define a business operation   | Domain       | `LoginUseCase`, `PlaceOrder`      |
| Define data contracts         | Domain       | `User` entity, `AuthRepository`   |
| Access API / database         | Data         | `AuthRepositoryImpl`, `ApiClient` |
| Map JSON to/from entities     | Data         | `UserModel`, `ProductDto`         |

## Feature Structure

```
lib/
├── core/
│   ├── errors/
│   │   ├── failures.dart
│   │   └── exceptions.dart
│   ├── network/
│   │   └── api_client.dart
│   ├── theme/
│   │   ├── app_theme.dart
│   │   └── app_colors.dart
│   └── routing/
│       └── app_router.dart
│
├── features/
│   ├── authentication/
│   │   ├── data/
│   │   │   ├── models/
│   │   │   │   └── user_model.dart
│   │   │   ├── datasources/
│   │   │   │   └── auth_remote_datasource.dart
│   │   │   └── repositories/
│   │   │       └── auth_repository_impl.dart
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   │   └── user.dart
│   │   │   ├── repositories/
│   │   │   │   └── auth_repository.dart
│   │   │   └── usecases/
│   │   │       ├── login.dart
│   │   │       └── signup.dart
│   │   └── presentation/
│   │       ├── pages/
│   │       │   └── login_page.dart
│   │       ├── widgets/
│   │       │   └── login_form.dart
│   │       └── controllers/
│   │           └── auth_controller.dart
│   │
│   └── products/
│       ├── data/
│       ├── domain/
│       └── presentation/
│
└── shared/
    ├── widgets/
    │   └── primary_button.dart
    └── extensions/
        └── context_extensions.dart
```

## Layer Details

### Domain Layer (`features/<feature>/domain/`)

**Purpose:** Business rules, entities, and repository contracts. Zero dependencies.

```dart
// domain/entities/user.dart
class User {
  final String id;
  final String email;
  final String name;

  const User({required this.id, required this.email, required this.name});
}

// domain/repositories/auth_repository.dart
abstract class AuthRepository {
  Future<User> login(String email, String password);
  Future<void> logout();
  Stream<User?> watchAuthState();
}

// domain/usecases/login.dart
class LoginUseCase {
  final AuthRepository _repository;
  LoginUseCase(this._repository);

  Future<User> call(String email, String password) {
    return _repository.login(email, password);
  }
}
```

**Rules:**
- ✅ Pure Dart only (no Flutter, no packages)
- ✅ Abstract repository interfaces
- ✅ Immutable entities
- ❌ No business logic in entities
- ❌ No framework imports

### Data Layer (`features/<feature>/data/`)

**Purpose:** Repository implementations, API/DB access, DTOs.

```dart
// data/models/user_model.dart
class UserModel {
  final String id;
  final String email;
  final String name;

  UserModel({required this.id, required this.email, required this.name});

  factory UserModel.fromJson(Map<String, dynamic> json) => UserModel(
    id: json['id'] as String,
    email: json['email'] as String,
    name: json['name'] as String,
  );

  User toEntity() => User(id: id, email: email, name: name);
}

// data/repositories/auth_repository_impl.dart
class AuthRepositoryImpl implements AuthRepository {
  final AuthRemoteDataSource _remote;
  AuthRepositoryImpl(this._remote);

  @override
  Future<User> login(String email, String password) async {
    final model = await _remote.login(email, password);
    return model.toEntity();
  }
}
```

**Rules:**
- ✅ Implements domain interfaces
- ✅ JSON serialization / deserialization
- ✅ Maps DTOs to domain entities
- ❌ No business logic
- ❌ No UI code

### Presentation Layer (`features/<feature>/presentation/`)

**Purpose:** UI, state management, user interaction.

```dart
// ❌ BAD: Business logic in widget
onPressed: () async {
  if (email.contains('@') && password.length >= 8) {
    final user = await apiClient.post('/login', ...);
    // ...
  }
}

// ✅ GOOD: Widget delegates to controller
onPressed: () => controller.login(email, password),
```

**Rules:**
- ✅ Widgets, pages, controllers
- ✅ Navigation and routing
- ✅ Depends on Domain only
- ❌ No direct API/DB calls
- ❌ No business rule validation

## When to Create a New Layer

| Scenario                                        | Action                                     |
|-------------------------------------------------|--------------------------------------------|
| Simple CRUD with no business rules              | Skip Domain, Data → Presentation directly  |
| Business logic shared across features           | Extract to Domain use case                 |
| Multiple data sources (API + cache)             | Full Data layer with repository pattern    |
| Feature used by other features                  | Extract shared entities to `core/domain/`  |
| Prototype / MVP                                 | Start simple, refactor layers when needed  |

## Error Handling Across Layers

```dart
// core/errors/failures.dart — Dart 3 sealed classes
sealed class Failure {
  const Failure([this.message]);
  final String? message;
}
class ServerFailure extends Failure { const ServerFailure([super.message]); }
class NetworkFailure extends Failure { const NetworkFailure([super.message]); }
class CacheFailure extends Failure { const CacheFailure([super.message]); }

// Data layer: catch exceptions, return failures
Future<Either<Failure, User>> login(String email, String password) async {
  try {
    final model = await _remote.login(email, password);
    return Right(model.toEntity());
  } on ServerException catch (e) {
    return Left(ServerFailure(e.message));
  }
}

// Presentation layer: pattern match on failure
final message = switch (failure) {
  ServerFailure(:final message) => message ?? 'Server error',
  NetworkFailure() => 'No internet connection',
  CacheFailure() => 'Local data unavailable',
};
```

## Common Violations

| Violation                        | Problem                   | Solution                                 |
|----------------------------------|---------------------------|------------------------------------------|
| Widget calls API directly        | Skips all layers          | Widget → Controller → UseCase → Repo    |
| Domain imports `package:http`    | Framework dependency      | Use abstract interface, implement in Data |
| Data layer has validation logic  | Business logic in wrong place | Move validation to Domain use case    |
| Feature A imports Feature B data | Tight coupling            | Extract shared code to `core/`           |
| Entity has `toJson()`           | Domain knows about serialization | Use separate DTO in Data layer      |

## Quick Checklist (New Feature)

- [ ] Domain entities are pure Dart with no imports?
- [ ] Repository interface defined in Domain?
- [ ] Repository implementation in Data?
- [ ] DTOs map to/from domain entities?
- [ ] Controller in Presentation only?
- [ ] Dependencies flow inward (Presentation → Domain ← Data)?
- [ ] Shared code extracted to `core/` or `shared/`?
- [ ] Feature folder is self-contained?
