# Flutter Architecture & Development Guidelines

This document outlines standard best practices, architectural principles, and performance guidelines for Flutter mobile application development.

---

## 1. Project Structure & Clean Architecture

Organize code by **Feature** rather than file type to ensure high modularity, maintainability, and scalability.

### Recommended Directory Layout

```text
lib/
├── app/
│   ├── config/             # App routing, themes, global constants
│   ├── observers/          # Bloc/Riverpod observers, navigation observers
│   └── service_locator.dart# Dependency injection configuration (GetIt)
├── core/
│   ├── constants/          # Asset paths, string constants, dimensions
│   ├── errors/             # Custom failures, exceptions, error handling
│   ├── network/            # HTTP clients (Dio), network info, interceptors
│   ├── utils/              # Extensions, formatters, validators
│   └── widgets/            # Generic/reusable UI elements (Buttons, Inputs)
├── features/
│   ├── authentication/
│   │   ├── data/           # Data models, DTOs, API providers, repository impls
│   │   ├── domain/         # Domain entities, use cases, repository interfaces
│   │   └── presentation/   # Screens, widgets, state management (Bloc/Riverpod)
│   └── profile/
└── main.dart               # App entry point
```

### Architectural Principles

* **Separation of Concerns:** Keep UI components, business logic, and API calls completely isolated from one another.
* **Domain-Driven Design (DDD):** Use domain entities and abstract interfaces so that business logic remains independent of external frameworks or libraries.
* **Code Generation:** Use packages such as `freezed` and `json_serializable` for immutable data models and robust JSON parsing.

---

## 2. State Management Guidelines

Avoid using `setState` at high levels of the widget tree, which triggers unnecessary full-screen rebuilds.

### Core Rules

1. **Establish a Standard:** Standardize state management across the team (**BLoC/Cubit** or **Riverpod** are recommended for production-grade applications).
2. **Localize Rebuild Scope:** Wrap only the specific widgets that depend on state changes (e.g., using `BlocBuilder` or `Consumer`).
3. **Immutability:** Ensure state classes are immutable. State should be updated by emitting or copying new state objects rather than mutating existing fields.

---

## 3. UI Performance & Rendering Optimization

### Leverage `const` Constructors

Mark every widget constructor as `const` wherever possible. This enables Flutter to cache widget subtrees and skip unnecessary rebuild cycles.

```dart
// GOOD
const Text('Submit', style: TextStyle(color: Colors.white));

// AVOID
Text('Submit', style: TextStyle(color: Colors.white));
```

### Optimize Lists and Dynamic Content

* **Lazy Loading:** Never use `Column` inside a `SingleChildScrollView` for long or dynamic dynamic lists. Always use `ListView.builder` or `GridView.builder` to dynamically construct items on screen.
* **Extract Widgets over Helper Functions:** Break complex screens down into small `StatelessWidget` classes instead of local UI functions (e.g., `_buildHeader()`). Classes benefit from framework-level rebuild optimizations, whereas functions rebuild unconditionally.

```dart
// GOOD: Extracted Widget Class
class UserCard extends StatelessWidget {
  final User user;
  const UserCard({super.key, required this.user});

  @override
  Widget build(BuildContext context) {
    return ListTile(title: Text(user.name));
  }
}

// AVOID: Helper Functions
Widget _buildUserCard(User user) {
  return ListTile(title: Text(user.name));
}
```

### Reduce Rendering Overhead

* Minimize heavy layout passes and operations like `Opacity`, `saveLayer()`, and aggressive `Clip` widgets.
* Use `FadeInImage` or explicit shape styling (`BoxDecoration.borderRadius`) instead of wrapping standard widgets in clipping or opacity layers.

---

## 4. Resource & Memory Management

### Dispose Controllers & Subscriptions

Always release resources in stateful lifecycle hooks to prevent memory leaks.

```dart
@override
void dispose() {
  _textController.dispose();
  _animationController.dispose();
  _streamSubscription.cancel();
  super.dispose();
}
```

### Asset & Image Optimization

* Pre-cache network images using `precacheImage()`.
* Specify `cacheWidth` and `cacheHeight` when rendering high-resolution images to limit RAM consumption.
* Use SVGs (via `flutter_svg`) for icons and vector graphics.

---

## 5. Security & Maintenance

### Secure Storage

* Never save sensitive user data, auth tokens, or private keys in standard `SharedPreferences`.
* Use `flutter_secure_storage` to encrypt data using iOS Keychain or Android Keystore.

### Strict Code Quality & Linting

Enforce strict linting rules (`flutter_lints` or `very_good_analysis`) across the project repository.

Recommended `analysis_options.yaml` settings:

```yaml
include: package:flutter_lints/flutter.yaml

linter:
  rules:
    - prefer_const_constructors
    - prefer_const_declarations
    - avoid_print
    - unawaited_futures
    - always_declare_return_types
```

---

## 6. Testing Strategy

Maintain an automated testing pyramid:

| Test Type | Target Scope | Key Tools |
| :--- | :--- | :--- |
| **Unit Tests** | Business logic, state controllers, repositories | `test`, `mocktail` |
| **Widget Tests** | Isolated UI components & interaction behavior | `flutter_test` |
| **Integration Tests** | End-to-end flows across the app | `integration_test` |

---

## Checklist for Code Review

- [ ] All UI logic is separated from business/network logic.
- [ ] No raw `print()` statements exist (use `log()` or a custom logging framework).
- [ ] `const` keywords are added where applicable.
- [ ] All controllers and streams are properly disposed of.
- [ ] Long lists use dynamic builders (`ListView.builder`).
- [ ] Sensitive data is stored in `flutter_secure_storage`.
- [ ] Unit/Widget tests are included for new features.