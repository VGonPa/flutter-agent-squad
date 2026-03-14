---
name: enforcing-flutter-standards
description: Enforces Flutter code quality standards while writing code. Use PROACTIVELY when creating Dart files, writing functions, implementing features, or reviewing code. Covers code style, naming conventions, import organization, const usage, widget extraction, analysis_options.yaml, custom lint rules, and file size management.
user-invocable: false
---

# Enforcing Flutter Standards

Active enforcement of Flutter code quality standards during development. Apply these rules proactively when writing or reviewing any Dart code.

## When to Use This Skill

Use **proactively** when:
- Creating new Dart files
- Writing functions, methods, or classes
- Implementing features or fixing bugs
- Adding providers, controllers, or widgets
- Reviewing code for quality issues

## Code Style and Naming Conventions

### Naming Rules

| Element | Convention | Example |
|---------|-----------|---------|
| Files | `snake_case` | `user_repository.dart` |
| Classes | `PascalCase` | `UserRepository` |
| Variables | `camelCase` | `currentUser` |
| Constants | `camelCase` | `defaultTimeout` |
| Private members | `_` prefix | `_internalState` |
| Enums | `PascalCase` name, `camelCase` values | `enum Status { active, inactive }` |
| Extensions | `PascalCase` on type | `extension StringX on String` |
| Typedefs | `PascalCase` | `typedef JsonMap = Map<String, dynamic>` |
| Test descriptions | lowercase sentence | `'should return user when found'` |

### Naming Smells to Avoid

```dart
// BAD: Vague names
var data = getData();
var temp = process(input);
Widget buildThing() => ...;

// GOOD: Descriptive names
var userProfile = getUserProfile();
var validatedInput = validateAndSanitize(rawInput);
Widget buildUserAvatar() => ...;
```

### Boolean Naming

```dart
// BAD
bool open = true;
bool data = false;

// GOOD: Use is/has/can/should prefix
bool isOpen = true;
bool hasData = false;
bool canSubmit = true;
bool shouldRefresh = false;
```

## Import Organization

**Strict order: Dart → Flutter → Packages → Project**

```dart
// 1. Dart SDK
import 'dart:async';
import 'dart:io';

// 2. Flutter SDK
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

// 3. External packages (alphabetical)
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

// 4. Project imports (relative within feature, package for cross-feature)
import '../model/user.dart';
import 'user_repository.dart';
```

**Rules:**
- Each group separated by blank line
- Alphabetical within each group
- Prefer relative imports within the same feature
- Use package imports for cross-feature references
- Never import `dart:mirrors`

## Const Usage

### Where to Use Const

```dart
// Constructors — mark const when all fields are final
class ProfileCard extends StatelessWidget {
  const ProfileCard({super.key, required this.name});
  final String name;
}

// Widget instances
const SizedBox(height: 16),
const EdgeInsets.all(8),
const Text('Static text'),
const Icon(Icons.check),

// Collections with known values
const colors = [Colors.red, Colors.blue];
const config = {'key': 'value'};
```

### When NOT to Use Const

```dart
// Dynamic values — cannot be const
SizedBox(height: spacing),   // Variable
Text(userName),               // Variable
Icon(iconData),               // Variable
```

**Rule**: Add `const` wherever the analyzer suggests it. Enable `prefer_const_constructors` and `prefer_const_literals_to_create_immutables` in analysis_options.

## Widget Best Practices

### Prefer Widget Classes Over Functions

```dart
// BAD: Widget function (no key, no const, no optimization)
Widget buildHeader() => Container(
  padding: const EdgeInsets.all(16),
  child: const Text('Header'),
);

// GOOD: Widget class (has key, can be const, rebuild boundary)
class _Header extends StatelessWidget {
  const _Header();

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(16),
      child: const Text('Header'),
    );
  }
}
```

### Widget Extraction Triggers

Extract a sub-widget when:
- A widget tree branch exceeds ~40 lines
- The same widget subtree appears in multiple places
- A section has its own distinct state or logic
- File is approaching the size limit

### Const Constructors in Widgets

```dart
// ALWAYS provide const constructors when possible
class MyWidget extends StatelessWidget {
  const MyWidget({super.key, required this.title});
  final String title;

  @override
  Widget build(BuildContext context) => Text(title);
}

// Use super.key instead of Key? key
// GOOD
const MyWidget({super.key});

// BAD (verbose)
const MyWidget({Key? key}) : super(key: key);
```

## File Size Management

### Target: 200-300 Lines Per File

Files exceeding 300 lines are hard to navigate and review. Strategies to keep files small:

**1. Extract Sub-Widgets**

```
// Before: single 350-line file
screens/profile_screen.dart (350 lines)

// After: split into focused files
screens/profile_screen.dart (80 lines)
widgets/profile_header.dart (70 lines)
widgets/profile_stats.dart (60 lines)
widgets/profile_actions.dart (50 lines)
```

**2. Split Large Services**

```
// Before: monolithic service
services/user_service.dart (400 lines)

// After: focused services
services/user_auth_service.dart (120 lines)
services/user_profile_service.dart (150 lines)
services/user_preferences_service.dart (100 lines)
```

**3. Move Constants and Configurations**

```dart
// Extract to dedicated file
// constants/profile_constants.dart
const maxBioLength = 300;
const defaultAvatarSize = 80.0;
const profilePadding = EdgeInsets.all(16);
```

**Note:** Dart uses `camelCase` for constants, not `kPrefix` (which is an Objective-C/iOS convention). See naming conventions table above.

**4. Separate Model Concerns**

```
// Split complex models
models/user.dart          // Core user model
models/user_stats.dart    // Stats sub-model
models/user_settings.dart // Settings sub-model
```

## analysis_options.yaml

### Recommended Configuration

```yaml
# analysis_options.yaml
include: package:flutter_lints/flutter.yaml

analyzer:
  errors:
    missing_return: error
    missing_required_param: error
    dead_code: warning
  exclude:
    - "**/*.g.dart"
    - "**/*.freezed.dart"
    - "lib/generated/**"

linter:
  rules:
    # Error prevention
    avoid_print: true
    avoid_relative_lib_imports: true
    avoid_returning_null_for_future: true
    avoid_slow_async_io: true
    cancel_subscriptions: true
    close_sinks: true
    no_duplicate_case_values: true

    # Style
    always_declare_return_types: true
    annotate_overrides: true
    avoid_empty_else: true
    avoid_init_to_null: true
    avoid_unnecessary_containers: true
    prefer_const_constructors: true
    prefer_const_constructors_in_immutables: true
    prefer_const_declarations: true
    prefer_const_literals_to_create_immutables: true
    prefer_final_fields: true
    prefer_final_locals: true
    prefer_if_null_operators: true
    prefer_single_quotes: true
    prefer_spread_collections: true
    sized_box_for_whitespace: true
    sort_child_properties_last: true
    unnecessary_brace_in_string_interps: true
    unnecessary_const: true
    unnecessary_new: true
    unnecessary_this: true
    use_key_in_widget_constructors: true

    # Documentation
    slash_for_doc_comments: true
    public_member_api_docs: false  # Enable for library packages

    # Dart 3 patterns
    use_enums: true
    use_super_parameters: true
```

### Custom Lint Rules with custom_lint

```yaml
# pubspec.yaml
dev_dependencies:
  custom_lint: ^0.6.0
  your_custom_lints: # your package or local rules
```

```yaml
# analysis_options.yaml
analyzer:
  plugins:
    - custom_lint
```

**Popular lint packages:**
- `flutter_lints` — Official Flutter team rules
- `very_good_analysis` — Strict rules from Very Good Ventures
- `lint` — Community-curated opinionated rules

## Error Handling Patterns

```dart
// In Controllers / Notifiers
Future<bool> submit(Data data) async {
  state = const AsyncValue.loading();
  state = await AsyncValue.guard(() async {
    await _service.process(data);
    return true;
  });
  return !state.hasError;
}

// In Services — log and rethrow
Future<Result> process(Data data) async {
  try {
    return await _repository.save(data);
  } catch (e, st) {
    _logger.severe('Process failed', e, st);
    rethrow;
  }
}

// In Repositories — return Either
Future<Either<Failure, Entity>> getEntity(String id) async {
  try {
    final doc = await _firestore.collection('entities').doc(id).get();
    if (!doc.exists) return Left(NotFoundFailure('Entity $id not found'));
    return Right(Entity.fromJson(doc.data()!));
  } catch (e) {
    return Left(ServerFailure(e.toString()));
  }
}
```

## Code Generation Files

```dart
// Always include part directives for generated code
part 'my_controller.g.dart';        // Riverpod
part 'user.freezed.dart';           // Freezed
part 'user.g.dart';                 // JSON serialization

// After ANY change to annotated code:
// dart run build_runner build --delete-conflicting-outputs
```

## Quick Checklist (Before Save)

- [ ] File under 300 lines?
- [ ] Imports in correct order (Dart → Flutter → Packages → Project)?
- [ ] Naming conventions followed?
- [ ] `const` used wherever possible?
- [ ] Widget classes instead of widget functions?
- [ ] `super.key` in widget constructors?
- [ ] Part directives for generated files?
- [ ] Error handling with AsyncValue.guard or Either?
- [ ] No `print()` statements (use proper logging)?
- [ ] No magic numbers (use named constants)?
- [ ] Boolean variables use is/has/can/should prefix?

## Common Violations Quick-Fix

| Violation | Fix |
|-----------|-----|
| `print()` in code | Use logging service or `debugPrint` |
| File >300 lines | Extract widgets, split services |
| Magic number spacing | Use named constants |
| Widget function | Extract to StatelessWidget |
| Missing part directive | Add `part 'file.g.dart';` |
| Unsorted imports | Reorder: Dart → Flutter → Packages → Project |
| Missing const | Add const to constructors and literals |
| `Key? key` in constructor | Use `super.key` |
| Vague variable name | Rename to describe content/purpose |
| Bare `catch (e) {}` | Log error, rethrow, or return failure |
