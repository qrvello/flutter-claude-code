# Flutter Performance: Profiling and Optimization Reference

Consolidated reference from the flutter-performance-analyzer and flutter-performance-optimizer agents.

## Table of Contents

- [Profiling with DevTools](#profiling-with-devtools)
  - [Launch and Connection](#launch-and-connection)
  - [Timeline / Performance Tab](#timeline--performance-tab)
  - [Performance Overlay](#performance-overlay)
  - [Memory Profiling](#memory-profiling)
  - [CPU Profiling](#cpu-profiling)
  - [Performance Targets](#performance-targets)
  - [Report Format](#report-format)
- [Optimization Techniques](#optimization-techniques)
  - [Const Constructors](#1-const-constructors)
  - [Widget Keys](#2-widget-keys)
  - [ListView.builder vs ListView](#3-listviewbuilder-vs-listview)
  - [RepaintBoundary](#4-repaintboundary)
  - [Image Optimization](#5-image-optimization)
  - [Isolates for Heavy Computation](#6-isolates-for-heavy-computation)
  - [Widget Tree Optimization](#7-widget-tree-optimization)
  - [Animation Optimization](#8-animation-optimization)
- [Common Anti-Patterns and Fixes](#common-anti-patterns-and-fixes)
- [Optimization Workflow](#optimization-workflow)

---

## Profiling with DevTools

### Launch and Connection

```bash
# From a running app -- press 'v' to open DevTools in browser
flutter run

# Standalone
flutter pub global activate devtools
flutter pub global run devtools

# From VS Code / Android Studio: Debug -> Open DevTools
# Manual connection: http://localhost:9100/?uri=http://localhost:xxxxx
```

### Timeline / Performance Tab

The Timeline shows per-frame rendering on two threads:

- **UI Thread** (blue bars) -- widget building
- **Raster Thread** (green bars) -- painting and compositing

Frame chart colors: green (< 16 ms), yellow (16-32 ms), red (> 32 ms / jank).

| Refresh Rate | Max Frame Time |
|---|---|
| 60 fps | 16 ms |
| 120 fps | 8 ms |

Red flags: consistent frames > 16 ms, spikes during interaction, tall UI bars (expensive `build()`), tall Raster bars (complex painting).

### Performance Overlay

```dart
MaterialApp(showPerformanceOverlay: true)
// Or: flutter run --profile --trace-skia
// Shows GPU (top) and UI (bottom) thread graphs -- both must stay below 16 ms.
```

Always profile in `--profile` mode. Debug mode distorts measurements.

### Memory Profiling

**Workflow:** snapshot before action, perform action, snapshot after, compare for retained objects / growth.

Look for: memory not released after navigating away, steady growth on repeated actions, large unexpected allocations.

**Common leak: stream listener never cancelled**

```dart
// LEAK
void initState() {
  super.initState();
  someStream.listen((data) => setState(() {})); // subscription leaks
}

// FIX
late final StreamSubscription _sub;
void initState() {
  super.initState();
  _sub = someStream.listen((data) => setState(() {}));
}
void dispose() { _sub.cancel(); super.dispose(); }
```

**Common leak: controller never disposed**

```dart
// LEAK
class _MyState extends State<MyWidget> {
  final controller = TextEditingController(); // never disposed
}

// FIX
void dispose() { controller.dispose(); super.dispose(); }
```

### CPU Profiling

| View | Purpose |
|---|---|
| **Call Tree** | Caller-to-callee chains -- understand flow |
| **Bottom Up** | Sorted by self-time -- find hottest methods |
| **Flame Chart** | Visual call stack (width = time, height = depth) |

Steps: record during slow operation, switch to Bottom Up, sort by Self Time, investigate methods > 100 ms.

```dart
// Custom timeline markers
import 'dart:developer' as developer;
developer.Timeline.startSync('expensive_operation');
// ... work ...
developer.Timeline.finishSync();
```

### Performance Targets

| Metric | Target |
|---|---|
| Frame time (60 fps) | < 16 ms |
| Jank rate | < 1% of frames |
| Memory (simple app) | < 100 MB, stable |
| Cold start | < 3 seconds |
| Simple widget build | < 1 ms |
| Complex screen build | < 50 ms |

### Report Format

```
# Performance Analysis Report
## Screen: [Name]
**Issue**: [user-visible symptom]
## Metrics
- Frame Rate: [current] (target: 60 fps)
- Jank Rate: [%] of frames > 16 ms
- Memory: [size]
## Findings
### Issue 1: [Title] (HIGH/MEDIUM/LOW)
**Location**: file.dart:line
**Impact**: [quantified]
**Evidence**: [DevTools observation]
**Recommendation**: [specific fix]
**Expected improvement**: [quantified]
## Priority Actions
1. [Fix] (priority) -- estimated time
```

---

## Optimization Techniques

### 1. Const Constructors

Const widgets are canonicalized at compile time and reused across frames, preventing unnecessary rebuilds.

```dart
// BEFORE: rebuilt every time parent rebuilds
class ProductCard extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Card(
      child: Column(children: [
        Icon(Icons.shopping_cart),  // rebuilt unnecessarily
        Text('Product'),
        SizedBox(height: 8),
      ]),
    );
  }
}

// AFTER: const constructor + const children
class ProductCard extends StatelessWidget {
  const ProductCard({super.key});
  @override
  Widget build(BuildContext context) {
    return Card(
      child: Column(children: const [
        Icon(Icons.shopping_cart),  // not rebuilt
        Text('Product'),
        SizedBox(height: 8),
      ]),
    );
  }
}
```

**Rule:** If a widget and all its properties are compile-time constants, make it `const`.

### 2. Widget Keys

Keys help Flutter correctly match stateful widgets across rebuilds, especially in reorderable lists.

```dart
// BEFORE: no keys -- Flutter may misidentify widgets after reorder
ListView.builder(
  itemBuilder: (context, index) => ProductCard(product: items[index]),
)

// AFTER: ValueKey for stable identity
ListView.builder(
  itemBuilder: (context, index) => ProductCard(
    key: ValueKey(items[index].id),
    product: items[index],
  ),
)
```

| Key Type | Use Case |
|---|---|
| `ValueKey` | Unique primitive (String, int) |
| `ObjectKey` | Unique complex object |
| `UniqueKey` | Truly unique each time (use sparingly) |
| `GlobalKey` | Access state across tree (expensive -- avoid in lists) |

### 3. ListView.builder vs ListView

`ListView.builder` lazily builds only visible items. For 10,000 items: build time drops from ~2000 ms to < 100 ms.

```dart
// BEFORE: all items built immediately
ListView(
  children: items.map((item) => ItemWidget(item)).toList(),
)

// AFTER: only visible items built
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) => ItemWidget(
    key: ValueKey(items[index].id),
    item: items[index],
  ),
)
```

Same principle for grids:

```dart
// BEFORE                              // AFTER
GridView.count(                        GridView.builder(
  crossAxisCount: 2,                     gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
  children: items.map(                     crossAxisCount: 2,
    (i) => ItemWidget(i)                 ),
  ).toList(),                            itemCount: items.length,
)                                        itemBuilder: (ctx, i) => ItemWidget(
                                           key: ValueKey(items[i].id),
                                           item: items[i],
                                         ),
                                       )
```

### 4. RepaintBoundary

Isolates a subtree so repaints inside it do not cascade to parent or siblings.

```dart
// Use around animated widgets, frequently-changing widgets, expensive CustomPaint
RepaintBoundary(child: AnimatedWidget())

RepaintBoundary(
  child: CustomPaint(
    painter: ComplexPainter(),
    child: SizedBox(width: 300, height: 300),
  ),
)
```

**Warning:** RepaintBoundary allocates an offscreen buffer. Only add it when the profiler confirms cascading repaints. Do not wrap simple widgets.

### 5. Image Optimization

Reduces memory 70-90% when display size is smaller than source size.

```dart
// BEFORE: full 4000x3000 decoded for 100x100 display
Image.network('https://example.com/image.jpg')

// AFTER: decode at 2x display size
Image.network(
  'https://example.com/image.jpg',
  cacheWidth: 200, cacheHeight: 200,
  fit: BoxFit.cover,
)

// BETTER: cached_network_image with placeholder
CachedNetworkImage(
  imageUrl: 'https://example.com/image.jpg',
  memCacheWidth: 200, memCacheHeight: 200,
  placeholder: (context, url) => const CircularProgressIndicator(),
  errorWidget: (context, url, error) => const Icon(Icons.error),
)

// BEST: responsive sizing
LayoutBuilder(builder: (context, constraints) {
  final size = (constraints.maxWidth * 2).toInt();
  return CachedNetworkImage(imageUrl: url, memCacheWidth: size, memCacheHeight: size);
})
```

### 6. Isolates for Heavy Computation

Keeps the UI thread free so frames render at 60 fps.

```dart
// BEFORE: UI freezes for 500 ms
void processLargeData() {
  final result = largeList.map((item) => expensiveTransform(item)).toList();
  setState(() => data = result);
}

// AFTER: compute() runs in a separate isolate
Future<void> processLargeData() async {
  final result = await compute(_processData, largeList);
  setState(() => data = result);
}
List<Result> _processData(List<Item> items) {
  return items.map((item) => expensiveTransform(item)).toList();
}
```

Also move expensive work out of `build()`:

```dart
// BEFORE: filtering on every build (50 ms each time)
@override
Widget build(BuildContext context) {
  final processed = items.where((i) => i.isActive).map(transform).toList();
  return ListView.builder(
    itemCount: processed.length,
    itemBuilder: (ctx, i) => ItemWidget(processed[i]),
  );
}

// AFTER: compute once in initState
late final List<Item> processed;
@override
void initState() {
  super.initState();
  processed = widget.items.where((i) => i.isActive).map(transform).toList();
}
@override
Widget build(BuildContext context) {
  return ListView.builder(
    itemCount: processed.length,
    itemBuilder: (ctx, i) => ItemWidget(processed[i]),
  );
}
```

### 7. Widget Tree Optimization

**Use Consumer/Selector to limit rebuild scope:**

```dart
// BEFORE: entire screen rebuilds on any cart change
Widget build(BuildContext context) {
  final cart = context.watch<CartProvider>();
  return Column(children: [
    AppBar(title: Text('Cart (${cart.itemCount})')),
    Expanded(child: CartList(cart.items)),
    CartSummary(cart.total),
    CheckoutButton(),
  ]);
}

// AFTER: only affected subtrees rebuild
Widget build(BuildContext context) {
  return Column(children: [
    AppBar(title: Selector<CartProvider, int>(
      selector: (ctx, cart) => cart.itemCount,
      builder: (ctx, count, _) => Text('Cart ($count)'),
    )),
    Expanded(child: Consumer<CartProvider>(
      builder: (ctx, cart, _) => CartList(cart.items),
    )),
    Consumer<CartProvider>(
      builder: (ctx, cart, _) => CartSummary(cart.total),
    ),
    const CheckoutButton(), // never rebuilds
  ]);
}
```

**Extract static content into const widgets:**

```dart
// BEFORE: static header rebuilt on every parent rebuild
Widget build(BuildContext context) {
  return Column(children: [
    Container(
      padding: EdgeInsets.all(16),
      child: Text('Header', style: TextStyle(fontSize: 24)),
    ),
    DynamicContent(),
  ]);
}

// AFTER: extracted to const widget
class StaticHeader extends StatelessWidget {
  const StaticHeader({super.key});
  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(16),
      child: const Text('Header', style: TextStyle(fontSize: 24)),
    );
  }
}
// Usage: const StaticHeader() -- not rebuilt
```

### 8. Animation Optimization

```dart
// BEFORE: ExpensiveWidget rebuilt 60 times/second
AnimatedBuilder(
  animation: controller,
  builder: (context, child) => Transform.rotate(
    angle: controller.value * 2 * pi,
    child: ExpensiveWidget(), // rebuilt every frame
  ),
)

// AFTER: child parameter -- ExpensiveWidget built once
AnimatedBuilder(
  animation: controller,
  child: const ExpensiveWidget(), // built once
  builder: (context, child) => Transform.rotate(
    angle: controller.value * 2 * pi,
    child: child, // reused every frame
  ),
)
```

For complex animations, wrap with `RepaintBoundary` to prevent repaints from propagating.

---

## Common Anti-Patterns and Fixes

| Anti-Pattern | Fix |
|---|---|
| `ListView()` with all children | `ListView.builder()` |
| Missing `const` on static widgets | Add `const` constructors and usage |
| `context.watch()` at top of large widget | `Consumer` / `Selector` at leaf nodes |
| Expensive computation in `build()` | Move to `initState()` or `compute()` |
| Undisposed controllers / subscriptions | `dispose()` / `cancel()` in `dispose()` |
| Full-size images for small displays | `cacheWidth` / `cacheHeight` |
| `GlobalKey` in list items | `ValueKey` or `ObjectKey` |
| `RepaintBoundary` on simple widgets | Remove -- only use when profiler confirms need |
| Heavy work on UI thread | `compute()` or `Isolate.spawn()` |
| Optimizing without profiling | Always profile first, then fix measured bottleneck |

---

## Optimization Workflow

1. **Profile first.** Run in `--profile` mode, use DevTools to find the actual bottleneck.
2. **High-impact fixes:** `ListView.builder`, `const` constructors, image `cacheWidth`/`cacheHeight`.
3. **Measure.** Re-run profiler, compare frame times, jank rate, memory.
4. **Medium-impact fixes:** widget keys, extract static content, move computation out of `build()`.
5. **Measure again.** Verify improvement, check for regressions.
6. **Targeted fixes:** `RepaintBoundary` where profiler shows cascading repaints, `Selector` for precise rebuilds, isolates for heavy work.
7. **Final verification.** Profile on lowest-end target device. Confirm 60 fps stable, < 1% jank.
