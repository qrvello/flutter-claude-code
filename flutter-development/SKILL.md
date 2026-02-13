---
name: flutter-development
description: Comprehensive Flutter app development covering UI design-to-implementation, Clean Architecture, BLoC state management, platform integration (iOS/Android), backend services (REST, Firebase, AWS, GraphQL), performance optimization, testing, and deployment. Use when building, architecting, testing, optimizing, or deploying Flutter applications, implementing native platform features, integrating backend services, or converting designs to pixel-perfect Flutter code.
---

# Flutter Development

End-to-end Flutter development skill covering the full app lifecycle: design analysis, architecture, implementation, backend integration, testing, performance optimization, and deployment.

## Core Workflow

1. **Identify the task domain** from the table below
2. **Read the relevant reference file** for domain-specific patterns and guidance
3. **Apply the patterns** following Flutter and Dart best practices
4. **Validate** with tests and performance checks

## Domain Reference Guide

| Task | Reference File | Key Topics |
|------|---------------|------------|
| Convert designs to Flutter code | [ui-design-implementation.md](references/ui-design-implementation.md) | Design analysis, widget selection, pixel-perfect code gen, visual comparison, iteration workflow |
| Manage simulators/emulators | [device-management.md](references/device-management.md) | iOS simctl, Android adb/emulator, screenshots, multi-device testing |
| Design app architecture | [architecture.md](references/architecture.md) | Clean Architecture, project structure, DI (GetIt/Riverpod), navigation (GoRouter/AutoRoute), Freezed |
| Implement state management | [state-management.md](references/state-management.md) | BLoC, Cubit, flutter_bloc widgets, hydrated_bloc, replay_bloc, bloc_concurrency |
| Integrate native features | [platform-integration.md](references/platform-integration.md) | MethodChannel, EventChannel, Pigeon, Swift/Kotlin, iOS/Android frameworks |
| Connect backend services | [backend-integration.md](references/backend-integration.md) | Dio/REST, Firebase, AWS Amplify, GraphQL |
| Optimize performance | [performance.md](references/performance.md) | DevTools profiling, const constructors, keys, RepaintBoundary, ListView.builder, memory |
| Write tests | [testing.md](references/testing.md) | Unit, widget, integration, BLoC tests, Mockito/Mocktail, golden tests |
| Deploy to stores | [deployment.md](references/deployment.md) | iOS App Store, Android Play Store, code signing, Fastlane CI/CD |

## Pattern Quick Reference

| Need | Reference File |
|------|---------------|
| UI components (cards, lists, forms) | [widget-patterns.md](references/widget-patterns.md) |
| Test templates and strategies | [testing-patterns.md](references/testing-patterns.md) |
| Pre-deploy performance checklist | [performance-checklist.md](references/performance-checklist.md) |
| Secure storage, API security, auth | [security-patterns.md](references/security-patterns.md) |
| Animations and transitions | [animation-patterns.md](references/animation-patterns.md) |

## Mandatory Practices

### Use Freezed for All Data Classes

All state classes, events, union types, and data models use Freezed:

```dart
@freezed
sealed class AuthState with _$AuthState {
  const factory AuthState.initial() = AuthStateInitial;
  const factory AuthState.authenticated({required User user}) = AuthStateAuthenticated;
  const factory AuthState.error({required String message}) = AuthStateError;
}
```

### Use Const Constructors Everywhere

```dart
// Always const for static widgets
const SizedBox(height: 16)
const Icon(Icons.home)
const EdgeInsets.all(16)
const Text('Static text')
```

### Follow Effective Dart Documentation

- Use `///` doc comments on all public APIs
- Start with single-sentence summary ending with a period
- Use third-person verbs ("Saves", "Deletes", "Returns")

### Material 3 Theming

```dart
MaterialApp(
  theme: ThemeData(
    useMaterial3: true,
    colorScheme: ColorScheme.fromSeed(seedColor: seedColor),
  ),
)

// Access theme values
Theme.of(context).colorScheme.primary
Theme.of(context).textTheme.titleLarge
```

## Design-to-Implementation Workflow

For converting designs to pixel-perfect Flutter code:

1. **Analyze design** - Identify widgets, create hierarchy, extract design tokens. See [ui-design-implementation.md](references/ui-design-implementation.md).
2. **Generate code** - Implement with proper const, keys, accessibility, responsive layouts.
3. **Capture screenshot** - Use device tools to screenshot running app. See [device-management.md](references/device-management.md).
4. **Compare visually** - Use ImageMagick to compare screenshot vs design, generate diff report.
5. **Iterate** - Apply fixes until fidelity >= 95%. Max 10 iterations.

## New Project Setup

For new Flutter projects, follow this sequence:

1. **Architecture** - Choose pattern (Clean Architecture recommended), set up project structure. See [architecture.md](references/architecture.md).
2. **State management** - Set up BLoC/Cubit with proper DI. See [state-management.md](references/state-management.md).
3. **Backend** - Integrate APIs (REST, Firebase, AWS, or GraphQL). See [backend-integration.md](references/backend-integration.md).
4. **UI** - Implement screens following design specs. See [ui-design-implementation.md](references/ui-design-implementation.md).
5. **Testing** - Write unit, widget, and integration tests. See [testing.md](references/testing.md).
6. **Performance** - Profile and optimize. See [performance.md](references/performance.md).
7. **Deploy** - Configure signing and publish. See [deployment.md](references/deployment.md).

## Key Package Versions

```yaml
dependencies:
  flutter_bloc: ^9.1.1
  freezed_annotation: ^2.4.0
  json_annotation: ^4.8.0
  dio: ^5.4.0
  go_router: ^13.0.0
  get_it: ^7.6.0
  injectable: ^2.3.0

dev_dependencies:
  bloc_test: ^10.0.0
  freezed: ^2.4.0
  json_serializable: ^6.7.0
  build_runner: ^2.4.0
  mockito: ^5.4.0
  mocktail: ^1.0.0
```
