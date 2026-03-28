---
description: Runs Flutter tests with coverage and reporting options
allowed-tools: Bash, Read
model: haiku
---

# Run Flutter Tests

Execute Flutter tests with various options.

## Parameters

From `$ARGUMENTS`:
- `all` or no argument → Run all tests
- `coverage` → Run with coverage report
- `<path>` → Run specific test file or directory
- `<pattern>` → Run tests matching name pattern

## Pre-flight Check

Verify `pubspec.yaml` exists:

```bash
test -f pubspec.yaml || echo "ERROR: Not in a Flutter project directory"
```

## Execution Options

### Run All Tests

```bash
flutter test --reporter expanded
```

### Run with Coverage

```bash
flutter test --coverage --reporter expanded

# Generate HTML report (if lcov tools installed)
if command -v genhtml &> /dev/null; then
  genhtml coverage/lcov.info -o coverage/html
  echo "Coverage report: coverage/html/index.html"
fi
```

### Run Specific Test File

```bash
flutter test test/path/to/specific_test.dart --reporter expanded
```

### Run Tests by Pattern

```bash
flutter test --name "pattern" --reporter expanded
```

### Run Tests in Directory

```bash
flutter test test/src/features/auth/ --reporter expanded
```

## Output Format

### On Success

```markdown
## Test Results

**Tests Run:** [count]
**Passed:** [count]
**Failed:** 0
**Time:** [duration]

All tests passed!
```

### On Failure

```markdown
## Test Results

**Tests Run:** [count]
**Passed:** [count]
**Failed:** [count]

### Failed Tests

1. **test_name**
   - File: `test/path/file_test.dart`
   - Error: [error message]
   - Line: [line number]

### Suggestions

- [How to investigate/fix]
```

### With Coverage

```markdown
## Test Results

**Coverage:** [percentage]%

### Low Coverage Files

- `lib/src/file.dart`: 45% — consider adding tests

### Coverage Targets

| Layer | Target | Status |
|-------|--------|--------|
| Services | >=90% | [met/not met] |
| Controllers | >=80% | [met/not met] |
| Repositories | >=80% | [met/not met] |
```

## Common Issues

### Tests Won't Run

```bash
flutter clean && flutter pub get
dart run build_runner build --delete-conflicting-outputs
```

### Mock Errors

```
MissingStubError: 'methodName'
→ Add when(() => mock.methodName(any())).thenReturn(value);
```

### Provider Not Found

```
ProviderNotFoundException
→ Wrap widget with ProviderScope in test
```

## Usage

```
/test                    # Run all tests
/test coverage           # Run with coverage
/test test/src/auth/     # Run auth tests
/test "should login"     # Run tests matching pattern
```
