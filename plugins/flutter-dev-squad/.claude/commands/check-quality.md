# Check Quality

Run the quality gate script on the current Flutter project.

## Usage

```
/check-quality                          # Full pre-merge checks
/check-quality pre-commit               # Fast checks (no coverage)
/check-quality --only tests,format      # Selective checks
/check-quality --test-dirs "test/unit/"  # Specific test directories
/check-quality --output json            # Machine-readable output
```

## Instructions

Run the quality-checks.sh script from the flutter-dev-squad plugin. The script is located at:

```bash
QUALITY_SCRIPT="${HOME}/.claude/skills/flutter-dev-squad/scripts/quality-checks.sh"
```

If the script is not found there, try:
```bash
QUALITY_SCRIPT="$(find "${HOME}/.claude/skills" -path "*/flutter-dev-squad/scripts/quality-checks.sh" -print -quit 2>/dev/null)"
```

Execute it from the current project root with any arguments the user passed:

```bash
"$QUALITY_SCRIPT" $ARGUMENTS
```

## After Running

- If **all checks pass** (exit 0): report the summary and congratulate
- If **checks fail** (exit 1): report which checks failed and offer to fix the issues automatically
- If `--output json` was used: parse the JSON and present a structured summary

## Available Checks

format, analyze, tests, coverage, linecount, secrets, deadcode, noprint, architecture, metrics

## Available Options

| Option | Description |
|--------|-------------|
| `pre-commit` | Fast mode (no coverage) |
| `pre-merge` | Full mode (default) |
| `--only CHECK,...` | Run only specified checks |
| `--skip CHECK,...` | Skip specified checks |
| `--test-dirs "dir1/ dir2/"` | Test only specific directories |
| `--paths "dir1/ dir2/"` | Scope file checks to directories |
| `--output json` | Machine-readable JSON output |
| `--timeout N` | Override test timeout (default: 300s) |
