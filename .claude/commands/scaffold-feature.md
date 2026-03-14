---
description: Creates Clean Architecture directory structure for a new Flutter feature
allowed-tools: Bash, Glob
model: haiku
---

# Scaffold Feature

Creates the directory structure for a new feature following Clean Architecture patterns.

## Parameters

- **Feature name** (required): from `$ARGUMENTS`, e.g., "notifications", "user_settings"

If not provided, ask for it.

## Pre-flight Checks

1. Verify `pubspec.yaml` exists in the current directory
2. Check the feature doesn't already exist in `lib/src/features/`
3. Validate naming is snake_case

## Directory Structure

```
lib/src/features/{feature_name}/
├── model/              # Domain entities, data classes
├── data/               # Repositories, data sources
├── application/        # Services, use cases
└── presentation/
    ├── controllers/    # State management (Riverpod)
    ├── pages/          # Full screen widgets
    └── widgets/        # Reusable UI components
```

## Execution

1. **Validate feature name**
   - Must be snake_case (lowercase, underscores only)
   - Must not already exist

2. **Create directories**
   ```bash
   mkdir -p lib/src/features/{feature_name}/{model,data,application,presentation/controllers,presentation/pages,presentation/widgets}
   ```

3. **Create corresponding test directories**
   ```bash
   mkdir -p test/src/features/{feature_name}/{model,data,application,presentation/controllers,presentation/widgets}
   ```

4. **Report success**

## On Success

```markdown
## Feature scaffolded: {feature_name}

**Directories created:**
```
lib/src/features/{feature_name}/
├── model/
├── data/
├── application/
└── presentation/
    ├── controllers/
    ├── pages/
    └── widgets/
```

**Test directories created:**
```
test/src/features/{feature_name}/
├── model/
├── data/
├── application/
└── presentation/
    ├── controllers/
    └── widgets/
```

**Next steps:**
1. Create domain models in `model/`
2. Create repository in `data/`
3. Create service in `application/`
4. Create controller in `presentation/controllers/`
5. Create UI in `presentation/pages/` and `widgets/`
```

## On Error

If feature already exists:
```markdown
Feature `{feature_name}` already exists at `lib/src/features/{feature_name}/`.

Did you mean to modify it? Use the appropriate agent instead.
```

## Usage

```
/scaffold-feature notifications
/scaffold-feature user_settings
/scaffold-feature workout_sharing
```
