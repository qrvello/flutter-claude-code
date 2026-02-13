# Flutter Development - Codex Skill

A comprehensive Codex skill for end-to-end Flutter app development, covering the full lifecycle from design analysis through production deployment.

## What This Skill Provides

This skill equips Codex with deep Flutter expertise across 9 domains:

| Domain | Description |
|--------|-------------|
| **UI Design & Implementation** | Convert Figma/screenshot designs to pixel-perfect Flutter code with automated visual comparison |
| **Device Management** | iOS simulator and Android emulator management, screenshots, multi-device testing |
| **Architecture** | Clean Architecture, project structure, Freezed data classes, dependency injection, navigation |
| **State Management** | BLoC/Cubit ecosystem with flutter_bloc widgets, persistence, undo/redo, event transformers |
| **Platform Integration** | MethodChannel, EventChannel, Pigeon code gen, Swift/Kotlin native features |
| **Backend Integration** | REST (Dio), Firebase, AWS Amplify, GraphQL with auth, caching, and real-time data |
| **Performance** | DevTools profiling, const constructors, widget optimization, memory management |
| **Testing** | Unit, widget, integration, BLoC tests, Mockito/Mocktail mocking, golden tests |
| **Deployment** | iOS App Store & Android Play Store: signing, TestFlight, Fastlane CI/CD |

Plus 5 pattern quick-reference files for widgets, testing, performance, security, and animations.

## Installation

Add this skill to your Codex project:

```
flutter-development/
```

## Structure

```
flutter-development/
├── SKILL.md                              # Core skill definition and workflow navigation
├── agents/
│   └── openai.yaml                       # Codex agent UI configuration
└── references/
    ├── ui-design-implementation.md       # Design analysis, code gen, visual comparison, iteration
    ├── device-management.md              # simctl, adb, emulator CLI commands
    ├── architecture.md                   # Clean Architecture, Freezed, DI, navigation
    ├── state-management.md               # BLoC ecosystem patterns
    ├── platform-integration.md           # Platform channels, Swift/Kotlin integration
    ├── backend-integration.md            # REST, Firebase, AWS, GraphQL
    ├── performance.md                    # Profiling and optimization
    ├── testing.md                        # Test patterns and mocking
    ├── deployment.md                     # iOS/Android store deployment
    ├── widget-patterns.md                # Cards, lists, forms, dialogs, responsive layouts
    ├── testing-patterns.md               # Test templates (unit, widget, BLoC, integration)
    ├── performance-checklist.md          # Pre-deployment performance checklist
    ├── security-patterns.md              # Secure storage, API security, encryption
    └── animation-patterns.md             # Implicit, explicit, page transitions
```

## Key Workflows

### Design to Implementation
1. Analyze design file (Figma export, screenshot)
2. Generate Flutter code with proper const, keys, accessibility
3. Capture screenshot via simulator/emulator
4. Compare visually using ImageMagick
5. Iterate until fidelity >= 95%

### New Project Setup
1. Set up Clean Architecture project structure
2. Configure BLoC state management with DI
3. Integrate backend services (REST/Firebase/AWS/GraphQL)
4. Implement UI screens from design specs
5. Write tests, optimize performance, deploy

## Mandatory Conventions

- **Freezed** for all data classes, state classes, and union types
- **Const constructors** on all static widgets
- **Material 3** theming via `ColorScheme.fromSeed`
- **Effective Dart** documentation (`///` doc comments on public APIs)
- **BLoC pattern** for state management (Cubit for simple, Bloc for complex)

## Origin

Converted from [flutter-claude-code](https://github.com/qrvello/flutter-claude-code), which contained 19 specialized Claude Code agents and 2 skills organized as modular plugins. All domain knowledge has been consolidated into this single Codex skill with organized reference files.
