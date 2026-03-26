# Technical Preferences

<!-- Populated by /setup-engine. Updated as the user makes decisions throughout development. -->
<!-- All agents reference this file for project-specific standards and conventions. -->

## Engine & Language

- **Engine**: Godot 4.6.1
- **Language**: GDScript (primary), C++ via GDExtension (performance-critical)
- **Rendering**: Godot 4.x Forward+/Mobile (Web 导出使用 WebGL)
- **Physics**: Godot Physics 3D/2D (内置)

## Naming Conventions

- **Classes**: PascalCase (e.g., `PlayerController`)
- **Variables**: snake_case (e.g., `move_speed`)
- **Signals/Events**: snake_case past tense (e.g., `health_changed`)
- **Files**: snake_case matching class (e.g., `player_controller.gd`)
- **Scenes/Prefabs**: PascalCase matching root node (e.g., `PlayerController.tscn`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `MAX_HEALTH`)

## Performance Budgets

- **Target Framerate**: 60 FPS
- **Frame Budget**: 16.6ms
- **Draw Calls**: [TO BE CONFIGURED — typical 2D target: <200]
- **Memory Ceiling**: [TO BE CONFIGURED — typical 2D target: <512MB]

## Testing

- **Framework**: GUT (Godot Unit Test)
- **Minimum Coverage**: [TO BE CONFIGURED — typical target: 60%+]
- **Required Tests**: Balance formulas, gameplay systems, networking (if applicable)

## Forbidden Patterns

<!-- Add patterns that should never appear in this project's codebase -->
- [None configured yet — add as architectural decisions are made]

## Allowed Libraries / Addons

<!-- Add approved third-party dependencies here -->
- [None configured yet — add as dependencies are approved]

## Architecture Decisions Log

<!-- Quick reference linking to full ADRs in docs/architecture/ -->
- [No ADRs yet — use /architecture-decision to create one]

## Localization (i18n)

- **Supported Languages**: 简体中文 (zh_CN), 日本語 (ja_JP), English (en_US)
- **Implementation**: Godot built-in i18n system (CSV/GetText)
- **Text Storage**: `assets/locale/` directory with CSV translation files
- **Font Strategy**: [TO BE CONFIGURED — recommend Noto Sans CJK for Chinese/Japanese]
- **RTL Support**: Not required (no RTL languages in target set)
