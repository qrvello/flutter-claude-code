# Flutter Architecture and Freezed Patterns Reference

Consolidated from the Flutter Architect and Freezed agent knowledge bases.

## Table of Contents

1. [Clean Architecture Layers](#clean-architecture-layers)
2. [Project Structure Template](#project-structure-template)
3. [Feature Module Layout](#feature-module-layout)
4. [Dependency Injection (GetIt)](#dependency-injection-getit)
5. [Navigation Architecture (GoRouter)](#navigation-architecture-gorouter)
6. [Freezed Setup](#freezed-setup)
7. [Freezed Core Patterns](#freezed-core-patterns)
8. [Union Types and Pattern Matching](#union-types-and-pattern-matching)
9. [JSON Serialization with Freezed](#json-serialization-with-freezed)
10. [build_runner Usage](#build_runner-usage)
11. [Code Documentation Standards](#code-documentation-standards)
12. [MVVM and MVI Alternatives](#mvvm-and-mvi-alternatives)

---

## Clean Architecture Layers

Dependencies flow inward: Presentation -> Domain -> Data. The **Presentation Layer** contains widgets, pages, and state management (BLoC/Cubit/Provider); it depends on Domain only. The **Domain Layer** contains entities, use cases, and repository interfaces; it depends on nothing. The **Data Layer** contains repository implementations, data sources (API, DB, cache), DTOs, and mappers; it depends on Domain interfaces.

---

## Project Structure Template

```
lib/
|-- core/
|   |-- constants/
|   |   |-- app_constants.dart
|   |   |-- api_constants.dart
|   |   +-- route_constants.dart
|   |-- errors/
|   |   |-- failures.dart
|   |   +-- exceptions.dart
|   |-- network/
|   |   |-- network_info.dart
|   |   +-- api_client.dart
|   |-- utils/
|   |   |-- validators.dart
|   |   |-- formatters.dart
|   |   +-- helpers.dart
|   |-- theme/
|   |   |-- app_theme.dart
|   |   |-- app_colors.dart
|   |   +-- app_text_styles.dart
|   +-- di/
|       +-- injection_container.dart
|
|-- features/
|   |-- authentication/
|   |   |-- data/
|   |   |   |-- models/
|   |   |   |   +-- user_model.dart
|   |   |   |-- datasources/
|   |   |   |   |-- auth_remote_datasource.dart
|   |   |   |   +-- auth_local_datasource.dart
|   |   |   +-- repositories/
|   |   |       +-- auth_repository_impl.dart
|   |   |-- domain/
|   |   |   |-- entities/
|   |   |   |   +-- user.dart
|   |   |   |-- repositories/
|   |   |   |   +-- auth_repository.dart
|   |   |   +-- usecases/
|   |   |       |-- login.dart
|   |   |       |-- logout.dart
|   |   |       +-- signup.dart
|   |   +-- presentation/
|   |       |-- pages/
|   |       |   |-- login_page.dart
|   |       |   +-- signup_page.dart
|   |       |-- widgets/
|   |       |   |-- login_form.dart
|   |       |   +-- social_login_buttons.dart
|   |       +-- bloc/
|   |           |-- auth_bloc.dart
|   |           |-- auth_event.dart
|   |           +-- auth_state.dart
|   |
|   |-- home/
|   |   |-- data/
|   |   |-- domain/
|   |   +-- presentation/
|   |
|   +-- products/
|       |-- data/
|       |-- domain/
|       +-- presentation/
|
+-- shared/
    |-- widgets/
    |   |-- buttons/
    |   |-- cards/
    |   +-- loading/
    +-- extensions/
        |-- string_extensions.dart
        +-- context_extensions.dart
```

- **`core/`** -- cross-cutting concerns: networking, errors, theming, DI, utilities.
- **`features/`** -- self-contained modules, each with `data/`, `domain/`, `presentation/`.
- **`shared/`** -- reusable widgets and extensions used across features.
- Features must not depend on each other directly. Shared domain concepts belong in `core/`.

---

## Feature Module Layout

```
features/products/
|-- data/
|   |-- models/
|   |   |-- product_model.dart
|   |   |-- product_model.freezed.dart
|   |   +-- product_model.g.dart
|   |-- datasources/
|   |   |-- product_remote_datasource.dart
|   |   +-- product_local_datasource.dart
|   +-- repositories/
|       +-- product_repository_impl.dart
|-- domain/
|   |-- entities/
|   |   +-- product.dart
|   |-- repositories/
|   |   +-- product_repository.dart        # abstract interface
|   +-- usecases/
|       |-- get_products.dart
|       +-- get_product_by_id.dart
+-- presentation/
    |-- pages/
    |-- widgets/
    +-- bloc/
        |-- products_bloc.dart
        |-- products_bloc.freezed.dart
        |-- products_event.dart
        +-- products_state.dart
```

Use barrel exports to simplify imports:

```dart
// features/products/products.dart
export 'domain/entities/product.dart';
export 'domain/usecases/get_products.dart';
export 'presentation/pages/products_page.dart';
export 'presentation/bloc/products_bloc.dart';
```

---

## Dependency Injection (GetIt)

```dart
// core/di/injection_container.dart
import 'package:get_it/get_it.dart';

final getIt = GetIt.instance;

Future<void> initializeDependencies() async {
  // Core
  getIt.registerLazySingleton(() => Dio()
    ..options.baseUrl = ApiConstants.baseUrl);
  getIt.registerLazySingleton<ApiClient>(() => ApiClient(getIt()));
  getIt.registerLazySingleton<NetworkInfo>(() => NetworkInfoImpl(getIt()));

  // Feature: Authentication
  getIt.registerLazySingleton<AuthRemoteDataSource>(
    () => AuthRemoteDataSourceImpl(client: getIt()),
  );
  getIt.registerLazySingleton<AuthRepository>(
    () => AuthRepositoryImpl(
      remoteDataSource: getIt(), localDataSource: getIt(), networkInfo: getIt(),
    ),
  );
  getIt.registerLazySingleton(() => Login(getIt()));
  getIt.registerFactory(() => AuthBloc(loginUseCase: getIt()));
}
```

Registration types: `registerSingleton` (immediate), `registerLazySingleton` (on first access), `registerFactory` (new instance each time), `registerFactoryParam` (with parameters).

```dart
// main.dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await initializeDependencies();
  runApp(const MyApp());
}
```

---

## Navigation Architecture (GoRouter)

```dart
// core/routing/app_router.dart
final appRouter = GoRouter(
  initialLocation: '/',
  routes: [
    GoRoute(path: '/', name: 'splash', builder: (_, __) => const SplashPage()),
    GoRoute(path: '/login', name: 'login', builder: (_, __) => const LoginPage()),
    GoRoute(
      path: '/home', name: 'home', builder: (_, __) => const HomePage(),
      routes: [
        GoRoute(
          path: 'products/:id', name: 'product-details',
          builder: (_, state) => ProductDetailsPage(
            productId: state.pathParameters['id']!,
          ),
        ),
      ],
    ),
  ],
  redirect: (context, state) {
    final isAuth = getIt<AuthBloc>().state is AuthStateAuthenticated;
    if (!isAuth && state.matchedLocation != '/login') return '/login';
    if (isAuth && state.matchedLocation == '/login') return '/home';
    return null;
  },
);

// Usage in MaterialApp
MaterialApp.router(routerConfig: appRouter);

// Navigation calls
context.go('/home/products/123');
context.goNamed('product-details', pathParameters: {'id': '123'});
context.pop();
```

---

## Freezed Setup

### Dependencies (pubspec.yaml)

```yaml
dependencies:
  freezed_annotation: ^2.4.0
  json_annotation: ^4.8.0
dev_dependencies:
  build_runner: ^2.4.0
  freezed: ^2.4.0
  json_serializable: ^6.7.0
```

### Analysis Options

```yaml
# analysis_options.yaml
analyzer:
  errors:
    invalid_annotation_target: ignore
  exclude:
    - "**/*.freezed.dart"
    - "**/*.g.dart"
```

### build.yaml (project-wide defaults)

```yaml
targets:
  $default:
    builders:
      freezed:
        options:
          format: true
          union_key: type
          union_value_case: pascal
      json_serializable:
        options:
          explicit_to_json: true
          include_if_null: false
          field_rename: snake
```

---

## Freezed Core Patterns

### Basic Immutable Data Class

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'product_model.freezed.dart';
part 'product_model.g.dart';

@freezed
class ProductModel with _$ProductModel {
  const factory ProductModel({
    required String id,
    required String name,
    required double price,
    @Default(true) bool isActive,
    @Default([]) List<String> tags,
  }) = _ProductModel;

  factory ProductModel.fromJson(Map<String, dynamic> json) =>
      _$ProductModelFromJson(json);
}
```

Generates: `copyWith`, `==`, `hashCode`, `toString`, `fromJson`, `toJson`.

### copyWith and Deep Copy

```dart
final product = ProductModel(id: '1', name: 'Widget', price: 9.99);
final updated = product.copyWith(price: 12.99);

// Deep copy for nested Freezed objects:
final updated = company.copyWith.director.assistant(name: 'Bob');
```

### Custom Methods (require private constructor)

```dart
@freezed
class Person with _$Person {
  const Person._();  // Required for custom methods

  const factory Person({
    required String firstName,
    required String lastName,
    required int age,
  }) = _Person;

  String get fullName => '$firstName $lastName';
  bool get isAdult => age >= 18;
  Person birthday() => copyWith(age: age + 1);

  factory Person.fromJson(Map<String, dynamic> json) => _$PersonFromJson(json);
}
```

### Assertions

```dart
@freezed
class Person with _$Person {
  @Assert('name.isNotEmpty', 'Name cannot be empty')
  @Assert('age >= 0', 'Age must be non-negative')
  const factory Person({required String name, required int age}) = _Person;
}
```

---

## Union Types and Pattern Matching

Used for BLoC states/events, error types, and exhaustive case handling.

### Defining Union Types

```dart
@freezed
sealed class AuthState with _$AuthState {
  const factory AuthState.initial() = AuthStateInitial;
  const factory AuthState.loading() = AuthStateLoading;
  const factory AuthState.authenticated({required User user}) = AuthStateAuthenticated;
  const factory AuthState.unauthenticated() = AuthStateUnauthenticated;
  const factory AuthState.error({required String message}) = AuthStateError;
}
```

Naming convention: variant classes use `{ClassName}{Variant}` (e.g., `AuthStateAuthenticated`).

### BLoC/Cubit with Freezed State

State files use `part of` the cubit/bloc file:

```dart
// auth_cubit.dart
part 'auth_cubit.freezed.dart';
part 'auth_state.dart';

class AuthCubit extends Cubit<AuthState> {
  AuthCubit({required LoginUseCase loginUseCase})
      : _loginUseCase = loginUseCase, super(const AuthState.initial());
  final LoginUseCase _loginUseCase;

  Future<void> login({required String email, required String password}) async {
    emit(const AuthState.loading());
    final result = await _loginUseCase(LoginParams(email: email, password: password));
    result.fold(
      (failure) => emit(AuthState.error(message: failure.message)),
      (user) => emit(AuthState.authenticated(user: user)),
    );
  }
}

// auth_state.dart
part of 'auth_cubit.dart';

@freezed
sealed class AuthState with _$AuthState {
  const factory AuthState.initial() = AuthStateInitial;
  const factory AuthState.loading() = AuthStateLoading;
  const factory AuthState.authenticated({required User user}) = AuthStateAuthenticated;
  const factory AuthState.unauthenticated() = AuthStateUnauthenticated;
  const factory AuthState.error({required String message}) = AuthStateError;
}
```

### Pattern Matching (Dart 3+, preferred)

```dart
final widget = switch (state) {
  AuthStateInitial() => const SizedBox.shrink(),
  AuthStateLoading() => const CircularProgressIndicator(),
  AuthStateAuthenticated(:final user) => Text('Hello, ${user.name}'),
  AuthStateUnauthenticated() => const LoginForm(),
  AuthStateError(:final message) => Text('Error: $message'),
};

if (state case AuthStateAuthenticated(:final user)) {
  navigateToHome(user);
}
```

### Legacy Pattern Matching (when/map)

```dart
state.when(
  initial: () => const SizedBox.shrink(),
  loading: () => const CircularProgressIndicator(),
  authenticated: (user) => Text('Hello, ${user.name}'),
  unauthenticated: () => const LoginForm(),
  error: (message) => Text('Error: $message'),
);

state.maybeWhen(
  authenticated: (user) => handleAuth(user),
  orElse: () => showLogin(),
);
```

### Error Handling with Union Types

```dart
@freezed
sealed class Failure with _$Failure {
  const factory Failure.server([String? message]) = FailureServer;
  const factory Failure.network([String? message]) = FailureNetwork;
  const factory Failure.cache([String? message]) = FailureCache;
  const factory Failure.validation([String? message]) = FailureValidation;
  const factory Failure.unexpected([String? message]) = FailureUnexpected;
}
```

---

## JSON Serialization with Freezed

### Basic

```dart
@freezed
class User with _$User {
  const factory User({required String id, required String email, String? name}) = _User;
  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
  // toJson() generated automatically
}
```

### @JsonKey

```dart
const factory Product({
  required String id,
  @JsonKey(name: 'unit_price') required double unitPrice,
  @JsonKey(name: 'is_active', defaultValue: true) bool isActive,
  @JsonKey(includeToJson: false) String? internalNote,
  @JsonKey(unknownEnumValue: Status.unknown) Status status,
}) = _Product;
```

### Generic Types

```dart
@Freezed(genericArgumentFactories: true)
class ApiResponse<T> with _$ApiResponse<T> {
  const factory ApiResponse({required T data, required bool success}) = _ApiResponse<T>;

  factory ApiResponse.fromJson(
    Map<String, dynamic> json, T Function(Object?) fromJsonT,
  ) => _$ApiResponseFromJson(json, fromJsonT);
}
```

### Custom JsonConverter

```dart
class DateTimeConverter implements JsonConverter<DateTime, String> {
  const DateTimeConverter();
  @override DateTime fromJson(String json) => DateTime.parse(json);
  @override String toJson(DateTime object) => object.toIso8601String();
}

// Usage: @DateTimeConverter() required DateTime createdAt,
```

### Union Type JSON

```dart
@Freezed(unionKey: 'type', unionValueCase: FreezedUnionCase.pascal)
sealed class Response with _$Response {
  @FreezedUnionValue('SuccessResponse')
  const factory Response.data(String value) = DataResponse;
  const factory Response.error(String message) = ErrorResponse;
  factory Response.fromJson(Map<String, dynamic> json) => _$ResponseFromJson(json);
}
// {"type": "SuccessResponse", "value": "hello"}
```

---

## build_runner Usage

```bash
# One-time build
dart run build_runner build --delete-conflicting-outputs

# Watch mode (recommended during development)
dart run build_runner watch -d

# Clean and rebuild
dart run build_runner clean && dart run build_runner build --delete-conflicting-outputs
```

### Common Errors

| Error | Cause | Fix |
|---|---|---|
| `Part file not found` | Generated files missing | Run build_runner build |
| `_$ClassName not defined` | Missing mixin or stale code | Ensure `class Foo with _$Foo`, clean + rebuild |
| `copyWith not defined` | IDE stale | Rebuild, restart Dart Analysis Server |
| `Conflicting outputs` | Old generated files | Use `--delete-conflicting-outputs` |
| Slow builds | Too many files | Add `generate_for` filters in `build.yaml` |

### Optimizing Build Scope

```yaml
# build.yaml
targets:
  $default:
    builders:
      freezed|freezed:
        generate_for:
          include:
            - lib/features/**/models/*.dart
            - lib/core/models/*.dart
          exclude:
            - "**/*.g.dart"
            - "**/*.freezed.dart"
```

---

## Code Documentation Standards

Follow Effective Dart. Use `///` for all doc comments. Start with a single-sentence summary ending with a period. Use `[brackets]` to link identifiers.

```dart
/// A repository that manages user data.
///
/// Coordinates between remote and local data sources.
class UserRepository {}

/// Saves the user to the database.
Future<void> saveUser(User user);

/// The user with the given [id], or null if not found.
User? getUserById(String id);

/// Whether the user is currently authenticated.
bool get isAuthenticated;

/// The current authenticated user.
final User currentUser;
```

---

## MVVM and MVI Alternatives

**MVVM**: ViewModel exposes observable state the View binds to (`ChangeNotifier` or Riverpod `Notifier`). Simpler than BLoC for smaller features -- no separate event classes needed. Flow: `View <--> ViewModel --> Model (Repository)`.

**MVI**: Unidirectional data flow -- View emits Intents, processor produces new Model state, View renders it. BLoC is already close to MVI (events = intents, states = model). Flow: `View --Intent--> Processor --Model--> View`.

Both patterns benefit from Clean Architecture layering and Freezed for immutable state classes.
