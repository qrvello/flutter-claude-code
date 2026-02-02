---
name: flutter-patterns
description: Use when implementing Flutter UI, writing tests, optimizing performance, securing data, or adding animations. Quick reference for production patterns.
---

# Flutter Patterns

Battle-tested patterns for production Flutter apps.

## Quick Reference

| Need | File | Key Patterns |
|------|------|--------------|
| UI components | flutter-widget-patterns.md | Cards, lists, forms, dialogs, navigation |
| Writing tests | flutter-testing-patterns.md | Unit, widget, BLoC, integration, mocking |
| Performance | flutter-performance-checklist.md | Build, image, list, memory optimization |
| Security | flutter-security-patterns.md | Storage, API, auth, encryption |
| Animations | flutter-animation-patterns.md | Implicit, explicit, page transitions |

## Pattern Categories

### Widget Patterns
See [patterns/flutter-widget-patterns.md](patterns/flutter-widget-patterns.md) for:
- Card patterns (basic, image, custom)
- List patterns (lazy loading, sectioned)
- Form patterns with validation
- Dialog and bottom sheet patterns
- Loading and empty states
- Navigation patterns
- Responsive layouts

### Testing Patterns
See [patterns/flutter-testing-patterns.md](patterns/flutter-testing-patterns.md) for:
- Unit test structure and best practices
- Widget testing approaches
- Integration test patterns
- Mock and stub strategies
- Test organization and naming conventions
- Coverage best practices

### Performance Patterns
See [patterns/flutter-performance-checklist.md](patterns/flutter-performance-checklist.md) for:
- Build method optimization
- Widget rebuilding minimization
- List and grid performance
- Image loading optimization
- Memory leak prevention
- Rendering performance tips

### Security Patterns
See [patterns/flutter-security-patterns.md](patterns/flutter-security-patterns.md) for:
- Input validation and sanitization
- Secure storage practices
- API security best practices
- Authentication and authorization patterns
- Data encryption approaches
- Common vulnerability prevention (XSS, injection, etc.)

### Animation Patterns
See [patterns/flutter-animation-patterns.md](patterns/flutter-animation-patterns.md) for:
- Basic animation controllers
- Tween animations
- Hero animations
- Page transitions
- Implicit animations
- Physics-based animations
- Custom animation patterns

## Usage Examples

**Example 1: Need a card UI pattern**
```
User: "I need to create a product card with an image and details"
→ Reference Widget Patterns for image card implementation
```

**Example 2: Writing tests**
```
User: "How should I structure my widget tests?"
→ Reference Testing Patterns for widget test examples
```

**Example 3: Performance issues**
```
User: "My list is laggy when scrolling"
→ Reference Performance Patterns for list optimization techniques
```

**Example 4: Security concern**
```
User: "How do I securely store user credentials?"
→ Reference Security Patterns for secure storage approaches
```

**Example 5: Adding animations**
```
User: "I want to animate a page transition"
→ Reference Animation Patterns for page transition examples
```

