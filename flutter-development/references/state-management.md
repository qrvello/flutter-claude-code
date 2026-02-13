# Flutter State Management with BLoC

## Table of Contents

1. [BLoC Package Ecosystem](#bloc-package-ecosystem)
2. [Cubit vs Bloc](#cubit-vs-bloc)
3. [BlocObserver for Debugging](#blocobserver-for-debugging)
4. [Cubit Implementation with Freezed](#cubit-implementation-with-freezed)
5. [Bloc Implementation with Freezed](#bloc-implementation-with-freezed)
6. [flutter_bloc Widgets](#flutter_bloc-widgets)
7. [RepositoryProvider for Dependency Injection](#repositoryprovider-for-dependency-injection)
8. [hydrated_bloc for State Persistence](#hydrated_bloc-for-state-persistence)
9. [replay_bloc for Undo/Redo](#replay_bloc-for-undoredo)
10. [bloc_concurrency Event Transformers](#bloc_concurrency-event-transformers)
11. [Testing with bloc_test](#testing-with-bloc_test)

---

## BLoC Package Ecosystem

| Package | Version | Purpose |
|---------|---------|---------|
| `bloc` | ^9.2.0 | Core: Cubit, Bloc, BlocObserver |
| `flutter_bloc` | ^9.1.1 | Widgets: BlocProvider, BlocBuilder, BlocListener, BlocConsumer, BlocSelector, RepositoryProvider |
| `bloc_test` | ^10.0.0 | Testing: blocTest(), MockBloc, MockCubit, whenListen |
| `hydrated_bloc` | ^10.1.1 | State persistence: HydratedBloc, HydratedCubit |
| `replay_bloc` | ^0.3.0 | Undo/redo: ReplayBloc, ReplayCubit |
| `bloc_concurrency` | ^0.3.0 | Event transformers: concurrent(), sequential(), droppable(), restartable() |

```yaml
dependencies:
  bloc: ^9.2.0
  flutter_bloc: ^9.1.1
  hydrated_bloc: ^10.1.1  # only if needed
  replay_bloc: ^0.3.0     # only if needed
dev_dependencies:
  bloc_test: ^10.0.0
  bloc_concurrency: ^0.3.0  # only if needed
```

---

## Cubit vs Bloc

| Use Case | Cubit | Bloc |
|----------|-------|------|
| Simple state changes (counter, toggle, form fields) | Yes | No |
| Complex event-driven logic | No | Yes |
| Need event traceability | No | Yes |
| Multiple events trigger same state | No | Yes |
| Search with debounce | No | Yes |

**Cubit**: Direct function calls -> `emit()` -> State.
**Bloc**: Events via `add()` -> `on<Event>` handlers -> `emit()` -> State.

---

## BlocObserver for Debugging

```dart
class AppBlocObserver extends BlocObserver {
  const AppBlocObserver();

  @override
  void onChange(BlocBase<dynamic> bloc, Change<dynamic> change) {
    super.onChange(bloc, change);
    debugPrint('onChange -- ${bloc.runtimeType}: $change');
  }

  @override
  void onTransition(Bloc<dynamic, dynamic> bloc, Transition<dynamic, dynamic> transition) {
    super.onTransition(bloc, transition);
    debugPrint('onTransition -- ${bloc.runtimeType}: $transition');
  }

  @override
  void onError(BlocBase<dynamic> bloc, Object error, StackTrace stackTrace) {
    debugPrint('onError -- ${bloc.runtimeType}: $error');
    super.onError(bloc, error, stackTrace);
  }
}

void main() {
  Bloc.observer = const AppBlocObserver();
  runApp(const MyApp());
}
```

Also supports `onCreate`, `onEvent`, and `onClose` overrides for full lifecycle visibility.

---

## Cubit Implementation with Freezed

State files use `part of` the cubit file. Two patterns are available.

### Pattern 1: Union Types (distinct states)

```dart
// auth_cubit.dart
part 'auth_cubit.freezed.dart';
part 'auth_state.dart';

class AuthCubit extends Cubit<AuthState> {
  AuthCubit({required AuthRepository repository})
      : _repository = repository, super(const AuthState.initial());
  final AuthRepository _repository;

  Future<void> login({required String email, required String password}) async {
    emit(const AuthState.loading());
    final result = await _repository.login(email: email, password: password);
    result.fold(
      (failure) => emit(AuthState.error(message: failure.message)),
      (user) => emit(AuthState.authenticated(user: user)),
    );
  }
}

// auth_state.dart -- part of 'auth_cubit.dart';
@freezed
sealed class AuthState with _$AuthState {
  const factory AuthState.initial() = AuthStateInitial;
  const factory AuthState.loading() = AuthStateLoading;
  const factory AuthState.authenticated({required User user}) = AuthStateAuthenticated;
  const factory AuthState.unauthenticated() = AuthStateUnauthenticated;
  const factory AuthState.error({required String message}) = AuthStateError;
}
```

### Pattern 2: Single State with Status Enum (shared fields)

```dart
// form_cubit.dart
class FormCubit extends Cubit<FormState> {
  FormCubit() : super(const FormState());
  void updateEmail(String email) => emit(state.copyWith(email: email));
  void updatePassword(String password) => emit(state.copyWith(password: password));
}

// form_state.dart -- part of 'form_cubit.dart';
enum FormStatus { initial, submitting, success, failure }

@freezed
abstract class FormState with _$FormState {
  const factory FormState({
    @Default(FormStatus.initial) FormStatus status,
    @Default('') String email,
    @Default('') String password,
    String? errorMessage,
  }) = _FormState;
}
```

---

## Bloc Implementation with Freezed

Both event and state files are `part of` the bloc file. Events use **past tense** naming.

```dart
// auth_bloc.dart
part 'auth_bloc.freezed.dart';
part 'auth_event.dart';
part 'auth_state.dart';

class AuthBloc extends Bloc<AuthEvent, AuthState> {
  AuthBloc({required LoginUseCase loginUseCase, required LogoutUseCase logoutUseCase})
      : _loginUseCase = loginUseCase, _logoutUseCase = logoutUseCase,
        super(const AuthState.initial()) {
    on<AuthEventLoginRequested>(_onLoginRequested);
    on<AuthEventLogoutRequested>(_onLogoutRequested);
  }
  final LoginUseCase _loginUseCase;
  final LogoutUseCase _logoutUseCase;

  Future<void> _onLoginRequested(AuthEventLoginRequested event, Emitter<AuthState> emit) async {
    emit(const AuthState.loading());
    final result = await _loginUseCase(LoginParams(email: event.email, password: event.password));
    result.fold(
      (failure) => emit(AuthState.error(message: failure.message)),
      (user) => emit(AuthState.authenticated(user: user)),
    );
  }

  Future<void> _onLogoutRequested(AuthEventLogoutRequested event, Emitter<AuthState> emit) async {
    emit(const AuthState.loading());
    await _logoutUseCase();
    emit(const AuthState.unauthenticated());
  }
}

// auth_event.dart -- part of 'auth_bloc.dart';
@freezed
sealed class AuthEvent with _$AuthEvent {
  const factory AuthEvent.loginRequested({required String email, required String password}) = AuthEventLoginRequested;
  const factory AuthEvent.logoutRequested() = AuthEventLogoutRequested;
}

// auth_state.dart -- part of 'auth_bloc.dart'; (same as Cubit example above)
```

**Naming conventions**: Events use past tense (`CounterIncrementPressed`, not `IncrementCounter`). States use `BlocSubject` + `State` (`AuthStateLoading`). Internal events use underscore prefix (`TimerEvent._ticked`).

---

## flutter_bloc Widgets

### BlocProvider and MultiBlocProvider

```dart
// Single provider (lazy by default)
BlocProvider(
  create: (context) => AuthBloc(repository: context.read<AuthRepository>()),
  child: const LoginPage(),
)

// Eager creation
BlocProvider(lazy: false, create: (context) => AuthBloc(...), child: const LoginPage())

// Existing instance (no auto-dispose)
BlocProvider.value(value: existingBloc, child: const ChildWidget())

// Multiple providers
MultiBlocProvider(
  providers: [
    BlocProvider<AuthBloc>(create: (context) => AuthBloc(repository: context.read<AuthRepository>())),
    BlocProvider<CartCubit>(create: (context) => CartCubit()),
  ],
  child: const MyApp(),
)
```

### BlocBuilder

```dart
BlocBuilder<CounterCubit, int>(
  builder: (context, count) => Text('Count: $count'),
)

// With buildWhen to limit rebuilds
BlocBuilder<AuthBloc, AuthState>(
  buildWhen: (previous, current) => previous is AuthStateLoading || current is AuthStateLoading,
  builder: (context, state) {
    return state.maybeWhen(
      loading: () => const CircularProgressIndicator(),
      orElse: () => const LoginForm(),
    );
  },
)
```

### BlocListener and MultiBlocListener

Executes side effects (navigation, dialogs, snackbars). Does **not** rebuild UI.

```dart
BlocListener<AuthBloc, AuthState>(
  listenWhen: (previous, current) => previous is! AuthStateError && current is AuthStateError,
  listener: (context, state) {
    state.maybeWhen(
      authenticated: (user) => Navigator.of(context).pushReplacementNamed('/home'),
      error: (message) => ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(message))),
      orElse: () {},
    );
  },
  child: const LoginPage(),
)

MultiBlocListener(
  listeners: [
    BlocListener<AuthBloc, AuthState>(listener: (context, state) { /* ... */ }),
    BlocListener<ConnectivityBloc, ConnectivityState>(listener: (context, state) { /* ... */ }),
  ],
  child: const AppShell(),
)
```

### BlocConsumer

Combines `BlocBuilder` and `BlocListener` into one widget.

```dart
BlocConsumer<AuthBloc, AuthState>(
  listenWhen: (previous, current) => current is AuthStateError,
  listener: (context, state) {
    state.maybeWhen(
      error: (message) => ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(message))),
      orElse: () {},
    );
  },
  buildWhen: (previous, current) => previous != current,
  builder: (context, state) {
    return state.maybeWhen(
      loading: () => const Center(child: CircularProgressIndicator()),
      orElse: () => const LoginForm(),
    );
  },
)
```

### BlocSelector

Selects a specific value from state; only rebuilds when that value changes.

```dart
BlocSelector<UserBloc, UserState, String>(
  selector: (state) => state.user.name,
  builder: (context, name) => Text('Hello, $name'),
)

// Tuple selection
BlocSelector<CartBloc, CartState, (int, double)>(
  selector: (state) => (state.itemCount, state.totalPrice),
  builder: (context, data) {
    final (count, total) = data;
    return Text('$count items - \$${total.toStringAsFixed(2)}');
  },
)
```

---

## RepositoryProvider for Dependency Injection

Injects non-bloc dependencies (repositories, services) into the widget tree.

```dart
MultiRepositoryProvider(
  providers: [
    RepositoryProvider<AuthRepository>(create: (context) => AuthRepositoryImpl()),
    RepositoryProvider<ProductRepository>(create: (context) => ProductRepositoryImpl()),
  ],
  child: MultiBlocProvider(
    providers: [
      BlocProvider<AuthBloc>(
        create: (context) => AuthBloc(repository: context.read<AuthRepository>()),
      ),
    ],
    child: const MaterialApp(home: HomePage()),
  ),
)

// Access in child widgets
final authRepository = context.read<AuthRepository>();
```

---

## hydrated_bloc for State Persistence

Automatically persists and restores state across app restarts. Requires `fromJson`/`toJson`.

### Setup

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  HydratedBloc.storage = await HydratedStorage.build(
    storageDirectory: kIsWeb
        ? HydratedStorageDirectory.web
        : HydratedStorageDirectory((await getApplicationDocumentsDirectory()).path),
  );
  runApp(const MyApp());
}
```

### HydratedCubit

```dart
class SettingsCubit extends HydratedCubit<SettingsState> {
  SettingsCubit() : super(const SettingsState());

  void toggleDarkMode() => emit(state.copyWith(isDarkMode: !state.isDarkMode));
  void setLocale(String locale) => emit(state.copyWith(locale: locale));

  @override
  SettingsState? fromJson(Map<String, dynamic> json) => SettingsState.fromJson(json);
  @override
  Map<String, dynamic>? toJson(SettingsState state) => state.toJson();
}

@freezed
abstract class SettingsState with _$SettingsState {
  const factory SettingsState({
    @Default(false) bool isDarkMode,
    @Default('en') String locale,
    @Default(true) bool notificationsEnabled,
  }) = _SettingsState;
  factory SettingsState.fromJson(Map<String, dynamic> json) => _$SettingsStateFromJson(json);
}
```

Good use cases: theme/locale preferences, shopping cart, auth tokens, draft form data.

---

## replay_bloc for Undo/Redo

Extends Bloc/Cubit with `undo()`, `redo()`, `canUndo`, `canRedo`, and `clearHistory()`.

```dart
class TextEditorCubit extends ReplayCubit<TextEditorState> {
  TextEditorCubit() : super(const TextEditorState());
  void updateText(String text) => emit(state.copyWith(text: text));
  void toggleBold() => emit(state.copyWith(isBold: !state.isBold));
  void clearAll() { clearHistory(); emit(const TextEditorState()); }
}

// UI usage
BlocBuilder<TextEditorCubit, TextEditorState>(
  builder: (context, state) {
    final cubit = context.read<TextEditorCubit>();
    return Scaffold(
      appBar: AppBar(actions: [
        IconButton(icon: const Icon(Icons.undo), onPressed: cubit.canUndo ? cubit.undo : null),
        IconButton(icon: const Icon(Icons.redo), onPressed: cubit.canRedo ? cubit.redo : null),
      ]),
      body: TextField(onChanged: cubit.updateText, maxLines: null, expands: true),
    );
  },
)
```

Good use cases: text editors, drawing apps, form wizards, photo editors, game moves.

---

## bloc_concurrency Event Transformers

| Transformer | Behavior | Use Case |
|-------------|----------|----------|
| `concurrent()` | Process all events in parallel | Analytics, logging |
| `sequential()` | Process one at a time, in order | Queue processing, uploads |
| `droppable()` | Ignore new events while processing | Form submission, button clicks |
| `restartable()` | Cancel in-progress, start new | Search, typeahead, autocomplete |

```dart
import 'package:bloc_concurrency/bloc_concurrency.dart';

// droppable: prevent duplicate form submissions
on<FormEventSubmitted>(_onSubmitted, transformer: droppable());

// restartable: cancel previous search when new query arrives
on<SearchEventQueryChanged>(_onQueryChanged, transformer: restartable());

// sequential: process queue items in order
on<QueueEventProcess>(_onProcess, transformer: sequential());

// concurrent: fire-and-forget analytics
on<AnalyticsEventTrack>(_onTrack, transformer: concurrent());
```

### restartable() with debounce (common search pattern)

```dart
class SearchBloc extends Bloc<SearchEvent, SearchState> {
  SearchBloc({required SearchRepository repository})
      : _repository = repository, super(const SearchState.initial()) {
    on<SearchEventQueryChanged>(_onQueryChanged, transformer: restartable());
  }
  final SearchRepository _repository;

  Future<void> _onQueryChanged(SearchEventQueryChanged event, Emitter<SearchState> emit) async {
    if (event.query.isEmpty) { emit(const SearchState.initial()); return; }
    emit(const SearchState.loading());
    await Future.delayed(const Duration(milliseconds: 300)); // debounce
    final results = await _repository.search(event.query);
    emit(SearchState.loaded(results: results));
  }
}
```

---

## Testing with bloc_test

### blocTest() Parameters

- **build**: Create the Bloc/Cubit (set up mocks here).
- **seed**: Set initial state before `act`.
- **act**: Dispatch events or call methods.
- **expect**: List of expected emitted states.
- **skip**: Number of initial states to skip.
- **wait**: Duration for async completion.
- **verify**: Additional assertions (e.g., verify mock calls).

### Complete Example

```dart
import 'package:bloc_test/bloc_test.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

class MockAuthRepository extends Mock implements AuthRepository {}

void main() {
  late AuthBloc authBloc;
  late MockAuthRepository mockRepository;

  setUp(() {
    mockRepository = MockAuthRepository();
    authBloc = AuthBloc(repository: mockRepository);
  });
  tearDown(() => authBloc.close());

  blocTest<AuthBloc, AuthState>(
    'emits [loading, authenticated] when login succeeds',
    build: () {
      when(() => mockRepository.login(any(), any())).thenAnswer((_) async => Right(testUser));
      return authBloc;
    },
    act: (bloc) => bloc.add(const AuthEvent.loginRequested(email: 'a@b.com', password: 'pass1234')),
    expect: () => [const AuthState.loading(), AuthState.authenticated(user: testUser)],
  );

  blocTest<AuthBloc, AuthState>(
    'emits [loading, unauthenticated] when logout requested from authenticated state',
    build: () {
      when(() => mockRepository.logout()).thenAnswer((_) async {});
      return authBloc;
    },
    seed: () => AuthState.authenticated(user: testUser),
    act: (bloc) => bloc.add(const AuthEvent.logoutRequested()),
    expect: () => [const AuthState.loading(), const AuthState.unauthenticated()],
  );

  blocTest<AuthBloc, AuthState>(
    'emits [loading, error] when login fails',
    build: () {
      when(() => mockRepository.login(any(), any()))
          .thenAnswer((_) async => Left(ServerFailure('Server error')));
      return authBloc;
    },
    act: (bloc) => bloc.add(const AuthEvent.loginRequested(email: 'a@b.com', password: 'pass1234')),
    expect: () => [const AuthState.loading(), const AuthState.error(message: 'Server error occurred')],
    verify: (_) {
      verify(() => mockRepository.login('a@b.com', 'pass1234')).called(1);
    },
  );
}
```

### MockBloc, MockCubit, and whenListen for Widget Tests

```dart
class MockAuthBloc extends MockBloc<AuthEvent, AuthState> implements AuthBloc {}

void main() {
  late MockAuthBloc mockAuthBloc;
  setUp(() => mockAuthBloc = MockAuthBloc());

  testWidgets('renders authenticated state', (tester) async {
    whenListen(
      mockAuthBloc,
      Stream.fromIterable([const AuthState.loading(), AuthState.authenticated(user: testUser)]),
      initialState: const AuthState.initial(),
    );

    await tester.pumpWidget(
      BlocProvider<AuthBloc>.value(value: mockAuthBloc, child: const MaterialApp(home: LoginPage())),
    );
    expect(find.byType(CircularProgressIndicator), findsOneWidget);
    await tester.pump();
    expect(find.text('Welcome, ${testUser.name}'), findsOneWidget);
  });

  testWidgets('shows error message on error state', (tester) async {
    when(() => mockAuthBloc.state).thenReturn(const AuthState.error(message: 'Login failed'));

    await tester.pumpWidget(
      BlocProvider<AuthBloc>.value(value: mockAuthBloc, child: const MaterialApp(home: LoginPage())),
    );
    expect(find.text('Login failed'), findsOneWidget);
  });
}
```
