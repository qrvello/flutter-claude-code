# Flutter Testing Reference

## Table of Contents

1. [Test Dependencies](#test-dependencies)
2. [Test Folder Structure](#test-folder-structure)
3. [Unit Tests](#unit-tests)
4. [Widget Tests](#widget-tests)
5. [Integration Tests](#integration-tests)
6. [BLoC Testing](#bloc-testing)
7. [Mocking with Mockito and Mocktail](#mocking)
8. [Test Organization](#test-organization)
9. [Coverage](#coverage)

---

## Test Dependencies

Add these to `pubspec.yaml` under `dev_dependencies`:

```yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  integration_test:
    sdk: flutter

  # Mocking
  mockito: ^5.4.0
  mocktail: ^1.0.0

  # BLoC testing
  bloc_test: ^9.1.0

  # Code generation for Mockito mocks
  build_runner: ^2.4.0

  # Golden tests
  golden_toolkit: ^0.15.0

  # Network mocking
  http_mock_adapter: ^0.6.0
```

Choose either Mockito or Mocktail for mocking (details in the [Mocking](#mocking) section). You do not need both.

---

## Test Folder Structure

```
test/
  unit/
    models/
      product_test.dart
    repositories/
      products_repository_test.dart
    use_cases/
      get_products_use_case_test.dart
    utils/
      validators_test.dart
  widget/
    pages/
      product_list_page_test.dart
    widgets/
      product_card_test.dart
    blocs/
      products_bloc_test.dart
  integration/
    app_test.dart
  mocks/
    mock_repositories.dart
    mock_services.dart
  fixtures/
    product.json
    products_list.json
  helpers/
    pump_app.dart
    test_helpers.dart
```

Mirror your `lib/` structure inside `test/unit/` and `test/widget/`. Shared mocks, fixtures, and helpers each get their own top-level directory under `test/`.

---

## Unit Tests

### Structure and the Arrange/Act/Assert Pattern

Every test body follows three phases:

```dart
test('description of expected behavior', () {
  // Arrange -- set up data, mocks, and preconditions
  final repository = MockRepository();
  when(repository.getData()).thenAnswer((_) async => testData);

  // Act -- execute the code under test
  final result = await useCase.call();

  // Assert -- verify outcomes
  expect(result, expectedResult);
  verify(repository.getData()).called(1);
});
```

### setUp and tearDown

Use `setUp` to initialize objects before each test and `tearDown` to clean up. Use `late` for fields that are assigned in `setUp`.

```dart
void main() {
  late ProductsRepositoryImpl repository;
  late MockProductsRemoteDataSource mockRemoteDataSource;
  late MockProductsLocalDataSource mockLocalDataSource;

  setUp(() {
    mockRemoteDataSource = MockProductsRemoteDataSource();
    mockLocalDataSource = MockProductsLocalDataSource();
    repository = ProductsRepositoryImpl(
      remoteDataSource: mockRemoteDataSource,
      localDataSource: mockLocalDataSource,
    );
  });

  tearDown(() {
    // Close streams, dispose controllers, etc.
  });
}
```

### Async Testing

Return a `Future` or mark the callback `async`. `flutter_test` will wait for it.

```dart
test('fetches products from remote source', () async {
  when(mockRemoteDataSource.getProducts())
      .thenAnswer((_) async => tProducts);

  final result = await repository.getProducts();

  expect(result, Right(tProducts));
});
```

### Model Serialization Template

```dart
group('Product', () {
  test('fromJson creates Product correctly', () {
    final json = {
      'id': '1',
      'name': 'Test Product',
      'price': 9.99,
      'description': 'A test product',
    };

    final product = Product.fromJson(json);

    expect(product.id, '1');
    expect(product.name, 'Test Product');
    expect(product.price, 9.99);
  });

  test('toJson converts Product correctly', () {
    final product = Product(id: '1', name: 'Test Product', price: 9.99);
    final json = product.toJson();

    expect(json['id'], '1');
    expect(json['price'], 9.99);
  });

  test('copyWith creates new instance with updated values', () {
    final product = Product(id: '1', name: 'Test Product', price: 9.99);
    final updated = product.copyWith(price: 19.99);

    expect(updated.price, 19.99);
    expect(product.price, 9.99); // original unchanged
  });
});
```

---

## Widget Tests

### pumpWidget and the Test Helper Extension

Wrap widgets in `MaterialApp` so they have access to `MediaQuery`, themes, and navigation. A helper extension keeps tests concise.

```dart
// test/helpers/pump_app.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

extension WidgetTesterX on WidgetTester {
  Future<void> pumpApp(
    Widget widget, {
    NavigatorObserver? navigatorObserver,
  }) async {
    await pumpWidget(
      MaterialApp(
        home: widget,
        navigatorObservers: [
          if (navigatorObserver != null) navigatorObserver,
        ],
      ),
    );
  }
}
```

### Finder Patterns

```dart
// By text
find.text('Test Product')

// By widget type
find.byType(ProductCard)

// By Key (preferred for interaction targets)
find.byKey(Key('login_button'))

// By icon
find.byIcon(Icons.shopping_cart)

// Containing text in RichText widgets
find.textContaining('laptop', findRichText: true)

// Count assertions
expect(find.byType(ProductCard), findsOneWidget);
expect(find.byType(ProductCard), findsNWidgets(3));
expect(find.byType(ProductCard), findsWidgets);      // at least one
expect(find.text('Gone'), findsNothing);
```

### Interaction Simulation

```dart
testWidgets('ProductCard tap triggers callback', (tester) async {
  bool tapped = false;
  final product = Product(id: '1', name: 'Test', price: 9.99);

  await tester.pumpApp(
    ProductCard(product: product, onTap: () => tapped = true),
  );

  await tester.tap(find.byType(ProductCard));
  await tester.pump(); // rebuild after tap

  expect(tapped, true);
});
```

Key interaction methods:

- `tester.tap(finder)` -- tap a widget
- `tester.enterText(finder, 'text')` -- type into a text field
- `tester.drag(finder, offset)` -- drag/swipe
- `tester.longPress(finder)` -- long press
- `tester.pump()` -- trigger a single frame rebuild
- `tester.pumpAndSettle()` -- pump until all animations finish

### Golden Tests

Golden tests capture a screenshot and compare future runs against it.

```dart
import 'package:golden_toolkit/golden_toolkit.dart';

testGoldens('ProductCard golden test', (tester) async {
  final product = Product(id: '1', name: 'Golden Product', price: 99.99);

  await tester.pumpWidgetBuilder(
    ProductCard(product: product),
    wrapper: materialAppWrapper(),
    surfaceSize: Size(400, 200),
  );

  await screenMatchesGolden(tester, 'product_card');
});
```

Update golden files when the UI intentionally changes:

```bash
flutter test --update-goldens
```

---

## Integration Tests

Integration tests run on a real device or emulator. They live in `integration_test/` at the project root (not inside `test/`).

### Setup

```dart
// integration_test/app_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:myapp/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('login and browse flow', (tester) async {
    app.main();
    await tester.pumpAndSettle();

    // Login
    await tester.enterText(find.byKey(Key('email_field')), 'test@example.com');
    await tester.enterText(find.byKey(Key('password_field')), 'password123');
    await tester.tap(find.byKey(Key('login_button')));
    await tester.pumpAndSettle();

    // Verify navigation to home
    expect(find.text('Welcome'), findsOneWidget);

    // Browse products
    await tester.tap(find.byIcon(Icons.shopping_bag));
    await tester.pumpAndSettle();
    expect(find.byType(ProductCard), findsWidgets);
  });
}
```

### Running Integration Tests

```bash
# Run on a connected device
flutter test integration_test/app_test.dart

# Run on a specific device
flutter test integration_test/app_test.dart -d <device-id>

# Run with the driver for reporting
flutter drive \
  --driver=test_driver/integration_test.dart \
  --target=integration_test/app_test.dart
```

---

## BLoC Testing

The `bloc_test` package provides `blocTest()`, a declarative way to test BLoC event-to-state mappings.

### blocTest Template

```dart
import 'package:bloc_test/bloc_test.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  late ProductsBloc bloc;
  late MockGetProducts mockGetProducts;

  setUp(() {
    mockGetProducts = MockGetProducts();
    bloc = ProductsBloc(getProducts: mockGetProducts);
  });

  tearDown(() => bloc.close());

  test('initial state is ProductsInitial', () {
    expect(bloc.state, ProductsInitial());
  });

  blocTest<ProductsBloc, ProductsState>(
    'emits [Loading, Loaded] when LoadProducts succeeds',
    build: () {
      when(mockGetProducts(any))
          .thenAnswer((_) async => Right(tProducts));
      return bloc;
    },
    act: (bloc) => bloc.add(LoadProducts()),
    expect: () => [
      ProductsLoading(),
      ProductsLoaded(tProducts),
    ],
    verify: (_) {
      verify(mockGetProducts(NoParams())).called(1);
    },
  );

  blocTest<ProductsBloc, ProductsState>(
    'emits [Loading, Error] when LoadProducts fails',
    build: () {
      when(mockGetProducts(any))
          .thenAnswer((_) async => Left(ServerFailure('Failed')));
      return bloc;
    },
    act: (bloc) => bloc.add(LoadProducts()),
    expect: () => [
      ProductsLoading(),
      ProductsError('Failed to load products'),
    ],
  );
}
```

### Key blocTest Parameters

| Parameter | Purpose |
|-----------|---------|
| `build`   | Create and return the BLoC instance. Set up mocks here. |
| `seed`    | Set the BLoC to a specific state before `act` runs. |
| `act`     | Add events to the BLoC. |
| `expect`  | List of states the BLoC should emit, in order. |
| `verify`  | Run additional verifications after the test (e.g., `verify` calls on mocks). |

### Using seed for Pre-set State

```dart
blocTest<ProductsBloc, ProductsState>(
  'emits filtered products when SearchProducts is added',
  build: () => bloc,
  seed: () => ProductsLoaded(tProducts),
  act: (bloc) => bloc.add(SearchProducts('Product 1')),
  expect: () => [
    ProductsLoaded([tProducts[0]]),
  ],
);
```

---

## Mocking

### Mockito (Code Generation)

Mockito requires `build_runner` to generate mock classes. Use `@GenerateMocks` to declare which classes to mock.

**Step 1: Declare mocks**

```dart
// test/mocks/mock_repositories.dart
import 'package:mockito/annotations.dart';
import 'package:myapp/data/datasources/products_remote_datasource.dart';
import 'package:myapp/data/datasources/products_local_datasource.dart';

@GenerateMocks([
  ProductsRemoteDataSource,
  ProductsLocalDataSource,
])
void main() {}
```

**Step 2: Generate**

```bash
flutter pub run build_runner build
```

This creates `mock_repositories.mocks.dart` with `MockProductsRemoteDataSource` and `MockProductsLocalDataSource`.

**Step 3: Use in tests**

```dart
import '../../mocks/mock_repositories.mocks.dart';

setUp(() {
  mockRemoteDataSource = MockProductsRemoteDataSource();
});

test('returns products', () async {
  when(mockRemoteDataSource.getProducts())
      .thenAnswer((_) async => tProducts);

  final result = await repository.getProducts();

  expect(result, Right(tProducts));
  verify(mockRemoteDataSource.getProducts()).called(1);
});
```

### Mocktail (No Code Generation)

Mocktail does not need `build_runner`. Define mocks inline with `extends Mock implements`.

**Step 1: Declare mocks**

```dart
// test/mocks/mock_repositories.dart
import 'package:mocktail/mocktail.dart';
import 'package:myapp/domain/repositories/user_repository.dart';

class MockUserRepository extends Mock implements UserRepository {}
```

**Step 2: Use in tests**

```dart
setUp(() {
  mockRepository = MockUserRepository();
});

test('getUser returns user', () async {
  // Note the lambda syntax: when(() => ...)
  when(() => mockRepository.getUser('123'))
      .thenAnswer((_) async => User(id: '123', name: 'Test'));

  final user = await mockRepository.getUser('123');

  expect(user.name, 'Test');
  verify(() => mockRepository.getUser('123')).called(1);
});
```

### Mockito vs Mocktail at a Glance

| Feature | Mockito | Mocktail |
|---------|---------|----------|
| Code generation required | Yes | No |
| Stub syntax | `when(mock.method())` | `when(() => mock.method())` |
| Verify syntax | `verify(mock.method())` | `verify(() => mock.method())` |
| Argument matchers | `any`, `argThat(...)` | `any()`, `any(that: ...)` |
| Null safety | Via codegen | Built-in |
| Setup overhead | Run `build_runner` after changes | None |

---

## Test Organization

### Naming Conventions

Test files mirror source files with a `_test.dart` suffix:

- `lib/models/product.dart` -> `test/unit/models/product_test.dart`
- `lib/pages/home_page.dart` -> `test/widget/pages/home_page_test.dart`

Test descriptions should state the expected behavior:

```dart
test('getProducts returns list when API call succeeds', () {});
test('getProducts throws ServerException when API call fails', () {});
test('getProducts returns cached data when offline', () {});
```

### Grouping with group()

```dart
void main() {
  group('ProductsRepository', () {
    group('getProducts', () {
      test('returns products when remote call is successful', () {});
      test('returns cached products when remote call fails', () {});
      test('returns ServerFailure when both remote and cache fail', () {});
    });

    group('getProductById', () {
      test('returns product when found', () {});
      test('returns NotFoundFailure when not found', () {});
    });
  });
}
```

### Shared Fixtures

Load JSON fixtures from files for consistent test data:

```dart
// test/fixtures/fixtures.dart
import 'dart:convert';
import 'dart:io';

String fixture(String name) {
  return File('test/fixtures/$name').readAsStringSync();
}

Map<String, dynamic> jsonFixture(String name) {
  return jsonDecode(fixture(name));
}
```

Usage:

```dart
test('fromJson creates Product from fixture', () {
  final json = jsonFixture('product.json');
  final product = Product.fromJson(json);
  expect(product.id, '1');
});
```

---

## Coverage

### Running Coverage

```bash
# Run all tests and collect coverage
flutter test --coverage

# Generate an HTML report (requires lcov)
genhtml coverage/lcov.info -o coverage/html

# Open the report
open coverage/html/index.html    # macOS
xdg-open coverage/html/index.html  # Linux
```

### Excluding Generated Files

Add exclusions in `analysis_options.yaml` so generated code does not pollute coverage numbers:

```yaml
analyzer:
  exclude:
    - "**/*.g.dart"
    - "**/*.freezed.dart"
    - "**/generated/**"
```

### Interpreting the Report

- **Line coverage**: percentage of executable lines that were reached during tests.
- **Branch coverage**: percentage of conditional branches (if/else, switch) that were exercised.
- Files highlighted in red have low coverage and are candidates for additional tests.
- Aim for high coverage on business logic (repositories, use cases, BLoCs) and critical UI flows. Generated code and simple data classes can have lower thresholds.
