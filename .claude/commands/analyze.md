---
description: Runs Flutter static analysis and formatting checks
allowed-tools: Bash, Read
model: haiku
---

# Analyze Flutter Code

Run static analysis, formatting checks, and linting on the Flutter codebase.

## Parameters

From `$ARGUMENTS`:
- No argument → Run full analysis
- `<path>` → Analyze specific file or directory
- `fix` → Run with auto-fix for fixable issues

## Execution

### Full Analysis

```bash
# Check formatting
dart format --set-exit-if-changed .

# Run Flutter analyzer
flutter analyze

# Run custom lint rules (if configured)
dart run custom_lint 2>/dev/null || true
```

### Analyze Specific Path

```bash
flutter analyze lib/src/features/auth/
```

### Auto-fix Issues

```bash
dart fix --apply
dart format .
flutter analyze
```

## Output Format

### Clean Analysis

```markdown
## Analysis Results

**Issues Found:** 0

Code is clean! No issues detected.
```

### With Issues

```markdown
## Analysis Results

**Errors:** [count]
**Warnings:** [count]
**Info:** [count]

### Errors (must fix)

| File | Line | Issue |
|------|------|-------|
| `lib/file.dart` | 42 | [description] |

### Warnings (should fix)

| File | Line | Issue |
|------|------|-------|
| `lib/file.dart` | 15 | [description] |

### Quick Fixes

```bash
dart fix --apply  # Auto-fix [X] issues
```

### Manual Fixes Required

1. **file.dart:42** - [how to fix]
```

## Common Issues

| Issue | Auto-fixable | Fix |
|-------|-------------|-----|
| Unused import | Yes | `dart fix --apply` |
| Missing return type | No | Add explicit return type |
| `print()` in code | No | Use a logger instead |
| Missing `const` | Yes | `dart fix --apply` |
| Unformatted code | Yes | `dart format .` |

## Usage

```
/analyze                 # Full analysis
/analyze lib/src/auth/   # Analyze auth feature
/analyze fix             # Run with auto-fix
```
