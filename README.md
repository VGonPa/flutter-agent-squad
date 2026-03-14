# Flutter Agent Squad

A squad of 8 specialized AI agents + 11 reusable skills + 3 commands for production-grade Flutter development. Covers architecture, UI, state management, testing, deployment, and code quality.

Framework-aware (Riverpod). Opinionated on Clean Architecture.

## When to Use This Plugin

Use it when working on any Flutter project. It helps with:

- Structuring features using Clean Architecture layers
- Writing widgets, controllers, repositories, and tests
- Reviewing code for quality, performance, and security
- Deploying to iOS App Store and Google Play
- Implementing state management with Riverpod

## Contents

### Agents

| Agent | Description | Auto-loaded Skills |
|-------|-------------|-------------------|
| `flutter-architect` | Designs app architecture, plans feature structure, makes architectural decisions | designing-flutter-architecture |
| `flutter-backend-developer` | Implements API integrations, repositories, data sources, Firebase | integrating-flutter-backend, securing-flutter-apps |
| `flutter-code-reviewer` | Reviews code for architecture compliance, quality, performance, security | enforcing-flutter-standards, designing-flutter-architecture, optimizing-flutter-performance, securing-flutter-apps |
| `flutter-debugger` | Diagnoses errors, analyzes tracebacks, implements minimal fixes | optimizing-flutter-performance, testing-flutter |
| `flutter-deployer` | Builds, signs, and deploys apps to App Store and Google Play | deploying-flutter-ios, deploying-flutter-android, securing-flutter-apps |
| `flutter-state-developer` | Implements state management with Riverpod | implementing-riverpod, enforcing-flutter-standards |
| `flutter-test-engineer` | Writes unit, widget, and integration tests with high coverage | testing-flutter, enforcing-flutter-standards |
| `flutter-ui-developer` | Creates Material Design 3 widgets, responsive layouts, animations | building-flutter-widgets, animating-flutter-ui, enforcing-flutter-standards |

### Skills

| Skill | Description |
|-------|-------------|
| `designing-flutter-architecture` | Clean Architecture layers, feature-based organization, dependency rules |
| `building-flutter-widgets` | Widget composition, Material Design 3, responsive layouts, theming, accessibility |
| `implementing-riverpod` | Riverpod 2.x with code generation, AsyncNotifier, provider types, testing |

| `testing-flutter` | Unit/widget/integration/golden tests, mocktail, coverage targets |
| `optimizing-flutter-performance` | Jank diagnosis, rebuild reduction, DevTools profiling, memory leaks |
| `securing-flutter-apps` | OWASP Mobile Top 10, secure storage, certificate pinning, input validation |
| `animating-flutter-ui` | Implicit/explicit animations, page transitions, Hero, Rive/Lottie |
| `integrating-flutter-backend` | REST/GraphQL, Firebase, repository pattern, error handling |
| `deploying-flutter-ios` | Code signing, TestFlight, App Store, Fastlane, privacy manifests |
| `deploying-flutter-android` | Keystores, Google Play Console, app bundles, staged rollouts |
| `enforcing-flutter-standards` | Code style, naming, imports, const usage, file size limits, custom lint rules |

### Commands

| Command | Description |
|---------|-------------|
| `/scaffold-feature` | Creates Clean Architecture directory structure for a new feature |
| `/analyze` | Runs Flutter static analysis and formatting checks |
| `/test` | Runs Flutter tests with coverage and reporting options |

## Agent-Skill Integration Map

Each agent preloads relevant skills automatically. You get the knowledge without doing anything extra.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        SKILL COVERAGE BY AGENT                          │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  enforcing-flutter-standards ←────┬── flutter-code-reviewer              │
│                                   ├── flutter-state-developer            │
│                                   ├── flutter-test-engineer              │
│                                   └── flutter-ui-developer               │
│                                                                          │
│  designing-flutter-architecture ←─┬── flutter-architect                  │
│                                   └── flutter-code-reviewer              │
│                                                                          │
│  optimizing-flutter-performance ←─┬── flutter-code-reviewer              │
│                                   └── flutter-debugger                   │
│                                                                          │
│  securing-flutter-apps ←──────────┬── flutter-backend-developer          │
│                                   ├── flutter-code-reviewer              │
│                                   └── flutter-deployer                   │
│                                                                          │
│  implementing-riverpod ←───────────── flutter-state-developer            │
│                                                                          │
│  testing-flutter ←────────────────┬── flutter-test-engineer              │
│                                   └── flutter-debugger                   │
│                                                                          │
│  building-flutter-widgets ←────────── flutter-ui-developer               │
│                                                                          │
│  animating-flutter-ui ←────────────── flutter-ui-developer               │
│                                                                          │
│  integrating-flutter-backend ←─────── flutter-backend-developer          │
│                                                                          │
│  deploying-flutter-ios ←───────────── flutter-deployer                   │
│                                                                          │
│  deploying-flutter-android ←───────── flutter-deployer                   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

## Architecture Reference

The plugin assumes Clean Architecture with feature-based organization:

```
lib/src/features/<feature>/
├── model/              # Domain entities, data classes (Freezed)
├── data/               # Repositories, data sources (Firebase, REST)
├── application/        # Services, use cases (business logic)
└── presentation/
    ├── controllers/    # State management (Riverpod AsyncNotifier)
    ├── pages/          # Full screen widgets
    └── widgets/        # Reusable UI components
```

**Key Rules:**

- Dependencies flow down: Presentation → Application → Data → Model
- Each layer only imports from layers below it
- The `model/` layer has zero dependencies on other layers
- Code generation required after model/provider changes (`build_runner`)

## Installation

```bash
# Clone or download the plugin
git clone <repository-url> flutter-dev-toolkit

# In your Flutter project's Claude Code config, add the plugin path
```

## Usage

### Using Agents

Agents activate automatically when relevant, or you can reference them explicitly:

```
"Create a repository for user preferences"
→ flutter-backend-developer activates

"Fix this widget test that's failing"
→ flutter-debugger activates

"Review the auth feature code"
→ flutter-code-reviewer activates

"Deploy the app to TestFlight"
→ flutter-deployer activates
```

### Using Commands

```bash
/scaffold-feature notifications   # Create feature directories
/test coverage                    # Run tests with coverage
/analyze fix                      # Run analysis with auto-fix
```

### Using Skills Directly

Skills load automatically through agents, but you can also reference them for guidance without code execution:

```
"How should I structure Riverpod providers?"
→ implementing-riverpod skill provides patterns

"What are the Clean Architecture layer rules?"
→ designing-flutter-architecture skill explains boundaries
```

## Requirements

- Flutter SDK 3.x
- Dart 3.x
- Claude Code CLI

## License

MIT
