# Mobile Dev Agency

A multi-plugin repository of specialized AI agent squads for mobile app development. Each plugin is an independent squad of agents and skills focused on a specific domain.

## Plugins

### `flutter-dev-squad`
**8 agents + 11 skills + 3 commands** for production-grade Flutter development.

Covers: architecture, UI widgets, state management (Riverpod), testing, performance, security, deployment (iOS/Android), backend integration, code standards.

[Full documentation →](plugins/flutter-dev-squad/README.md)

### `mobile-design-squad`
**2 agents + 5 skills** for mobile UI/UX design.

Covers: design discovery, UX patterns (Laws of UX, navigation, forms), visual design (color, typography, spacing, animation, anti-AI-slop), Flutter UI implementation, design review (Nielsen heuristics, WCAG 2.2, delight audit).

[Full documentation →](plugins/mobile-design-squad/README.md)

## Installation

Each plugin can be installed independently:

```bash
# Clone the repo
git clone https://github.com/VGonPa/mobile-dev-agency.git

# Symlink the plugin(s) you want into your Claude Code skills directory
ln -s /path/to/mobile-dev-agency/plugins/flutter-dev-squad ~/.claude/skills/flutter-dev-squad
ln -s /path/to/mobile-dev-agency/plugins/mobile-design-squad ~/.claude/skills/mobile-design-squad
```

## Architecture

```
mobile-dev-agency/
├── plugins/
│   ├── flutter-dev-squad/        # Flutter development
│   │   ├── .claude/
│   │   │   ├── agents/           # 8 specialized agents
│   │   │   ├── skills/           # 11 skills (SKILL.md + REFERENCE.md)
│   │   │   └── commands/         # 3 slash commands
│   │   └── .claude-plugin/
│   └── mobile-design-squad/      # Mobile UI/UX design
│       ├── .claude/
│       │   ├── agents/           # 2 design agents
│       │   └── skills/           # 5 design skills
│       └── .claude-plugin/
└── README.md
```

## Requirements

- Claude Code CLI
- Flutter SDK 3.x / Dart 3.x (for flutter-dev-squad)

## License

MIT
