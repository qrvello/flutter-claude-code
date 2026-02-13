# Flutter Testing Patterns - Quick Reference

Production-ready test templates for comprehensive test coverage.

## Table of Contents

- [Unit Test Templates](#unit-test-templates)
- [Widget Test Templates](#widget-test-templates)
- [BLoC Testing Templates](#bloc-testing-templates)
- [Mock Patterns](#mock-patterns)
- [Integration Test Templates](#integration-test-templates)
- [Golden Test Templates](#golden-test-templates)
- [Test Helpers and Matchers](#test-helpers-and-matchers)

## Unit Test Templates

### Basic Unit Test

```dart
import 'package:flutter_test/flutter_test.dart';

void main() {
  group('ClassName', () {
    test('description of what is being tested', () {
      // Arrange
      final input = 'test input';
      final expected = 'expected output';
      // Act
      final result = functionToTest(input);
      // Assert
      expect(result, expected);
    });
  });
}
```

### Test with Setup and Teardown

```dart
void main() {
  late MyClass instance;
  setUp(() { instance = MyClass(); });
  tearDown(() { instance.dispose(); });
  test('test description', () { expect(instance.value, isNotNull); });
}
```

### Testing Streams

```dart
test('stream emits correct values', () {
  final controller = StreamController<int>();
  expect(controller.stream, emitsInOrder([1, 2, 3, emitsDone]));
  controller.add(1);
  controller.add(2);
  controller.add(3);
  controller.close();
});
```

## Widget Test Templates

### Basic Widget Test

```dart
testWidgets('widget description', (WidgetTester tester) async {
  await tester.pumpWidget(MaterialApp(home: MyWidget()));
  expect(find.text('Expected Text'), findsOneWidget);
});
```

### Test with BLoC Provider

```dart
testWidgets('widget with provider', (tester) async {
  final mockBloc = MockMyBloc();
  when(() => mockBloc.state).thenReturn(InitialState());
  await tester.pumpWidget(
    MaterialApp(
      home: BlocProvider<MyBloc>.value(value: mockBloc, child: MyWidget()),
    ),
  );
  expect(find.byType(MyWidget), findsOneWidget);
});
```

### Test User Interaction

```dart
testWidgets('button tap triggers callback', (tester) async {
  bool tapped = false;
  await tester.pumpWidget(MaterialApp(
    home: Scaffold(body: ElevatedButton(onPressed: () => tapped = true, child: Text('Tap me'))),
  ));
  await tester.tap(find.text('Tap me'));
  await tester.pump();
  expect(tapped, true);
});
```

### Test Text Input

```dart
testWidgets('text field accepts input', (tester) async {
  final controller = TextEditingController();
  await tester.pumpWidget(MaterialApp(
    home: Scaffold(body: TextField(controller: controller, key: Key('email_field'))),
  ));
  await tester.enterText(find.byKey(Key('email_field')), 'test@example.com');
  await tester.pump();
  expect(controller.text, 'test@example.com');
});
```

### Test Navigation

```dart
testWidgets('navigation to detail page', (tester) async {
  await tester.pumpWidget(MaterialApp(
    home: HomePage(),
    routes: {'/details': (context) => DetailsPage()},
  ));
  await tester.tap(find.text('View Details'));
  await tester.pumpAndSettle();
  expect(find.byType(DetailsPage), findsOneWidget);
});
```

## BLoC Testing Templates

### Basic BLoC Test

```dart
import 'package:bloc_test/bloc_test.dart';

void main() {
  group('MyBloc', () {
    late MyBloc bloc;
    setUp(() { bloc = MyBloc(); });
    tearDown(() { bloc.close(); });

    test('initial state is correct', () {
      expect(bloc.state, isA<InitialState>());
    });

    blocTest<MyBloc, MyState>(
      'emits [LoadingState, LoadedState] when LoadEvent is added',
      build: () => bloc,
      act: (bloc) => bloc.add(LoadEvent()),
      expect: () => [LoadingState(), LoadedState(data: testData)],
    );
  });
}
```

### BLoC Test with Mock Dependencies

```dart
blocTest<ProductsBloc, ProductsState>(
  'emits [Loading, Loaded] when LoadProducts succeeds',
  build: () {
    final mockRepository = MockProductsRepository();
    when(() => mockRepository.getProducts()).thenAnswer((_) async => Right(testProducts));
    return ProductsBloc(repository: mockRepository);
  },
  act: (bloc) => bloc.add(LoadProducts()),
  expect: () => [ProductsLoading(), ProductsLoaded(testProducts)],
  verify: (bloc) { verify(() => mockRepository.getProducts()).called(1); },
);
```

## Mock Patterns

### Mockito Setup

```dart
import 'package:mockito/annotations.dart';

@GenerateMocks([UserRepository, AuthService])
void main() {}
// Run: flutter pub run build_runner build
```

### Mocktail Pattern (No Code Gen)

```dart
import 'package:mocktail/mocktail.dart';

class MockUserRepository extends Mock implements UserRepository {}

void main() {
  late MockUserRepository mockRepo;
  setUp(() { mockRepo = MockUserRepository(); });

  test('using mocktail', () async {
    when(() => mockRepo.getUser(any())).thenAnswer((_) async => User(id: '1', name: 'Test'));
    final user = await mockRepo.getUser('1');
    expect(user.name, 'Test');
    verify(() => mockRepo.getUser(any())).called(1);
  });
}
```

## Integration Test Templates

### Basic Integration Test

```dart
import 'package:integration_test/integration_test.dart';
import 'package:myapp/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('full app test', (tester) async {
    app.main();
    await tester.pumpAndSettle();
    await tester.enterText(find.byKey(Key('email')), 'test@example.com');
    await tester.enterText(find.byKey(Key('password')), 'password123');
    await tester.tap(find.byKey(Key('login_button')));
    await tester.pumpAndSettle();
    expect(find.text('Welcome'), findsOneWidget);
  });
}
```

## Golden Test Templates

```dart
import 'package:golden_toolkit/golden_toolkit.dart';

testGoldens('widget golden test', (tester) async {
  await tester.pumpWidgetBuilder(MyWidget(), wrapper: materialAppWrapper(theme: ThemeData.light()), surfaceSize: Size(400, 600));
  await screenMatchesGolden(tester, 'my_widget');
});

testGoldens('responsive layout golden test', (tester) async {
  await tester.pumpWidgetBuilder(MyResponsiveWidget(), wrapper: materialAppWrapper());
  await multiScreenGolden(tester, 'responsive_widget', devices: [Device.phone, Device.iphone11, Device.tabletPortrait]);
});
```

## Test Helpers and Matchers

### Pump Widget Helper

```dart
extension WidgetTesterX on WidgetTester {
  Future<void> pumpApp(Widget widget) async {
    await pumpWidget(MaterialApp(home: widget));
  }
}
```

### Common Matchers

```dart
// Types & equality
expect(value, equals(expected));
expect(value, isA<MyClass>());
expect(value, isNull / isNotNull);

// Numbers
expect(value, greaterThan(5));
expect(value, closeTo(5.0, 0.1));

// Collections
expect(list, isEmpty / isNotEmpty / hasLength(3));
expect(list, contains(item));

// Widgets
expect(find.text('Hello'), findsOneWidget);
expect(find.byType(MyWidget), findsNWidgets(3));
expect(find.byKey(Key('key')), findsNothing);

// Exceptions
expect(() => fn(), throwsA(isA<MyException>()));

// Async
await expectLater(stream, emitsInOrder([1, 2, 3]));
```

## Best Practices

1. **AAA Pattern**: Arrange, Act, Assert
2. **One assertion per test**: Focus on single behavior
3. **Descriptive names**: Test names describe what they test
4. **Mock external dependencies**: Isolate unit under test
5. **Don't test implementation details**: Test behavior, not internals
6. **Clean up resources**: Dispose controllers, close streams
7. **Test edge cases**: Empty lists, null values, errors
8. **Golden tests for UI**: Catch visual regressions
9. **Integration tests for flows**: Test complete user journeys
