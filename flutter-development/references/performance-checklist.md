# Flutter Performance Checklist

Complete pre-deployment checklist to ensure optimal app performance.

## Build Performance

### Release Build Configuration

- [ ] Using `flutter build --release` (not debug)
- [ ] ProGuard/R8 enabled for Android (minifyEnabled true)
- [ ] Code shrinking enabled (shrinkResources true)
- [ ] Obfuscation configured
- [ ] No print statements (use logger with levels)

### App Size Optimization

- [ ] APK/AAB size < 20MB (ideal)
- [ ] Remove unused assets
- [ ] Use vector graphics (SVG) when possible
- [ ] Split APKs per ABI (`--split-per-abi`)
- [ ] Use deferred loading for large features

```dart
import 'package:heavy_feature.dart' deferred as heavy;
Future<void> loadHeavyFeature() async {
  await heavy.loadLibrary();
  heavy.showHeavyWidget();
}
```

## Widget Performance

### Const Constructors

- [ ] All static widgets use `const`
- [ ] Text, Icons, Padding/SizedBox with fixed values are const

```dart
// Bad                    // Good
Text('Static text')       const Text('Static text')
Icon(Icons.home)          const Icon(Icons.home)
SizedBox(height: 16)      const SizedBox(height: 16)
```

### List Performance

- [ ] Using `ListView.builder` (not ListView with children)
- [ ] `GridView.builder` for grids
- [ ] Infinite scroll uses pagination
- [ ] List items have keys (when reorderable)
- [ ] Complex list items use `RepaintBoundary`
- [ ] `itemExtent` specified (when items same height)

```dart
ListView.builder(
  itemExtent: 60,
  itemCount: items.length,
  itemBuilder: (context, index) {
    return RepaintBoundary(
      key: ValueKey(items[index].id),
      child: ItemWidget(items[index]),
    );
  },
)
```

### Image Optimization

- [ ] Images resized to display dimensions
- [ ] Using `cacheWidth`/`cacheHeight` for network images
- [ ] Placeholder for slow-loading images
- [ ] Error handling for failed image loads

```dart
Image.network(
  imageUrl,
  cacheWidth: 300, cacheHeight: 300,
  fit: BoxFit.cover,
  loadingBuilder: (context, child, progress) {
    if (progress == null) return child;
    return CircularProgressIndicator();
  },
  errorBuilder: (context, error, stackTrace) => Icon(Icons.error),
)
```

### Widget Tree Optimization

- [ ] No deep nesting (max 4-5 levels)
- [ ] Extract complex widgets into separate classes (not functions)
- [ ] Use `AnimatedOpacity` instead of `Opacity`
- [ ] `RepaintBoundary` for expensive widgets

## State Management Performance

- [ ] Using const constructors to prevent rebuilds
- [ ] Provider/BLoC listeners scoped appropriately
- [ ] Using `BlocSelector` for specific field updates
- [ ] No `setState` called in loops
- [ ] Heavy computations moved outside `build()`

```dart
// Good - Only rebuild on itemCount change
BlocSelector<CartBloc, CartState, int>(
  selector: (state) => state.itemCount,
  builder: (context, itemCount) => Text('$itemCount'),
)
```

## Computation Optimization

- [ ] Expensive calculations cached/memoized
- [ ] Heavy operations run in isolates
- [ ] Synchronous operations < 16ms

```dart
Future<List<Product>> filterProducts(List<Product> products) async {
  return await compute(_filterProducts, products);
}
```

## Animation Performance

- [ ] Animations run at 60fps (frames < 16ms)
- [ ] Using `AnimatedBuilder` for custom animations
- [ ] Complex animations use `RepaintBoundary`
- [ ] All animation controllers disposed
- [ ] Avoid animating layout (use `Transform` instead)

## Network Performance

- [ ] Implementing pagination for large datasets
- [ ] Request debouncing for search
- [ ] Caching API responses
- [ ] Retry logic with exponential backoff
- [ ] Timeout configurations (5-10 seconds)

## Memory Management

- [ ] Controllers disposed properly
- [ ] Stream subscriptions cancelled
- [ ] Listeners removed in `dispose()`
- [ ] Image cache size limited

## Profiling

```bash
flutter run --profile                    # Performance profiling
flutter run --profile --trace-skia       # With Skia tracing
flutter build apk --analyze-size         # Analyze app size
```

- [ ] Timeline shows no jank (all frames < 16ms)
- [ ] Memory stable (no continuous growth)
- [ ] Cold start < 3 seconds
- [ ] Scrolling smooth at 60fps

## Quick Performance Wins

1. Add `const` everywhere possible
2. Use `ListView.builder` instead of `ListView`
3. Optimize images with `cacheWidth`/`cacheHeight`
4. Move heavy computation to isolates
5. Add `RepaintBoundary` to expensive widgets
6. Implement pagination for lists
7. Cache network responses
8. Dispose all controllers and subscriptions
9. Test on real devices, not just simulators
