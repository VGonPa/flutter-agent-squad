---
name: optimizing-flutter-performance
description: Optimizes Flutter app performance. Use when diagnosing jank, reducing rebuilds, optimizing lists/images, profiling with DevTools, detecting memory leaks, or improving startup time. Covers Impeller, RepaintBoundary, const optimization, and common anti-patterns.
---

# Optimizing Flutter Performance

Performance optimization patterns, profiling techniques, and anti-pattern detection for Flutter apps.

## When to Use This Skill

- App has jank (dropped frames) or slow scrolling
- Diagnosing excessive widget rebuilds
- Optimizing list rendering or image loading
- Profiling with Flutter DevTools
- Detecting and fixing memory leaks
- Improving app startup time

## Performance Targets

| Metric | Target | Red Flag |
|--------|--------|----------|
| Frame time (60fps) | <16ms | >32ms |
| Frame time (120fps) | <8ms | >16ms |
| Jank rate | <1% of frames | >5% |
| App memory | <150MB simple apps | Continuous growth |
| Cold start | <3 seconds | >5 seconds |
| Widget build | <1ms simple widget | >50ms |

## Build Optimization

### Use `const` Everywhere Possible

```dart
// BAD: Rebuilds every time parent rebuilds
SizedBox(height: 16),
Text('Static text'),
EdgeInsets.all(8),

// GOOD: Compile-time constant, never rebuilt
const SizedBox(height: 16),
const Text('Static text'),
const EdgeInsets.all(8),

// GOOD: Const constructor enables const usage
class MyWidget extends StatelessWidget {
  const MyWidget({super.key, required this.title});
  final String title;
}
```

**Impact:** Const widgets are canonicalized - Flutter skips rebuild entirely when parent rebuilds.

### RepaintBoundary

```dart
// Isolate frequently-changing widgets from static subtrees
RepaintBoundary(
  child: AnimatedWidget(),  // Only this repaints
)

// Good candidates for RepaintBoundary:
// - Animations running independently
// - Scrolling content
// - Video players
// - Complex custom painters
```

**When NOT to use:** Don't wrap every widget. Each RepaintBoundary allocates a separate layer, consuming GPU memory. Profile first.

### Keys for Efficient Rebuilds

```dart
// Use ValueKey for lists with stable identifiers
ListView.builder(
  itemBuilder: (context, index) {
    return ProductCard(
      key: ValueKey(products[index].id),
      product: products[index],
    );
  },
)

// Use UniqueKey when you want to force recreation
AnimatedWidget(key: UniqueKey())  // New instance every rebuild
```

### Avoid Unnecessary Rebuilds

```dart
// BAD: Creates new list every build
Widget build(BuildContext context) {
  final items = List.generate(1000, (i) => Item(i)); // Expensive!
  return ListView(children: items.map(ItemWidget.new).toList());
}

// GOOD: Compute once, cache result
final _items = List.generate(1000, (i) => Item(i));

Widget build(BuildContext context) {
  return ListView.builder(
    itemCount: _items.length,
    itemBuilder: (_, i) => ItemWidget(item: _items[i]),
  );
}
```

## List Optimization

### Always Use ListView.builder for Long Lists

```dart
// BAD: Builds ALL 10,000 items immediately
ListView(
  children: items.map((item) => ItemWidget(item)).toList(),
)

// GOOD: Builds only visible items (lazy)
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) => ItemWidget(item: items[index]),
)

// BETTER: With fixed extent for even faster scrolling
ListView.builder(
  itemCount: items.length,
  itemExtent: 72.0, // Fixed height = O(1) scroll offset calculation
  itemBuilder: (context, index) => ItemWidget(item: items[index]),
)
```

### SliverList for Mixed Content

```dart
CustomScrollView(
  slivers: [
    SliverAppBar(title: Text('Products')),
    SliverList.builder(
      itemCount: products.length,
      itemBuilder: (context, index) => ProductCard(product: products[index]),
    ),
  ],
)
```

## Image Optimization

### Resize Images to Display Size

```dart
// BAD: Decodes 4000x3000 image for 100x100 display
Image.network(url)

// GOOD: Decode at display resolution (2x for retina)
Image.network(
  url,
  cacheWidth: 200,   // 100 * 2 for retina
  cacheHeight: 200,
  fit: BoxFit.cover,
)
```

### Use cached_network_image

```dart
CachedNetworkImage(
  imageUrl: url,
  memCacheWidth: 200,
  placeholder: (_, __) => const ShimmerPlaceholder(),
  errorWidget: (_, __, ___) => const Icon(Icons.error),
)
```

### Precache Critical Images

```dart
@override
void didChangeDependencies() {
  super.didChangeDependencies();
  precacheImage(AssetImage('assets/hero.png'), context);
}
```

## Heavy Computation

### Move to Isolates

```dart
// BAD: Blocks UI thread
void processData() {
  final result = List.generate(1000000, (i) => i * i); // 500ms freeze!
  setState(() => data = result);
}

// GOOD: Runs in separate isolate
void processData() async {
  final result = await compute(expensiveOperation, inputData);
  setState(() => data = result);
}

// Must be a top-level or static function (not a closure)
// Returns sync type — compute() handles the Future wrapping
static List<int> expensiveOperation(List<int> input) {
  return input.map((i) => i * i).toList();
}
```

## Impeller Renderer

Impeller is Flutter's modern rendering engine replacing Skia:

- **iOS**: Default since Flutter 3.16
- **Android**: Default since Flutter 3.38
- **Key benefit**: Eliminates shader compilation jank (all shaders pre-compiled at build time)
- **Result**: Smooth animations from the very first frame

```bash
# Profile with Impeller (default on modern Flutter)
flutter run --profile

# Compare with legacy Skia renderer
flutter run --profile --no-enable-impeller
```

**Impact on profiling:** If you still see first-frame jank with Impeller, it's widget building or data loading, not shaders.

## DevTools Profiling

### Launch DevTools

```bash
flutter run --profile
# Press 'v' to open DevTools in browser
```

### Timeline Analysis

```
Frame Time Chart:
  Green bars: Good (<16ms)
  Yellow bars: Warning (16-32ms)
  Red bars: Jank (>32ms)

  UI Thread (top): Widget building
  Raster Thread (bottom): Painting

Common Patterns:
  Tall UI bars -> Expensive build() methods
  Tall Raster bars -> Complex painting operations
  Both tall -> Overall performance issue
```

### Performance Overlay

```dart
MaterialApp(
  showPerformanceOverlay: true,
)
```

### Custom Timeline Markers

```dart
import 'dart:developer' as developer;

developer.Timeline.startSync('expensive_operation');
// ... operation ...
developer.Timeline.finishSync();
```

## Memory Leak Detection

### Common Leaks

**Listeners not removed:**
```dart
// BAD: Listener leaks on widget dispose
void initState() {
  super.initState();
  someStream.listen((data) => setState(() {}));
}

// GOOD: Cancel subscription
late StreamSubscription _subscription;

void initState() {
  super.initState();
  _subscription = someStream.listen((data) => setState(() {}));
}

void dispose() {
  _subscription.cancel();
  super.dispose();
}
```

**Controllers not disposed:**
```dart
// BAD: Controllers leak
class _MyState extends State<MyWidget> {
  final controller = TextEditingController();
  final scrollController = ScrollController();
  // Missing dispose()!
}

// GOOD: Always dispose controllers
void dispose() {
  controller.dispose();
  scrollController.dispose();
  super.dispose();
}
```

**AnimationController not disposed:**
```dart
// ALWAYS dispose in dispose()
late AnimationController _controller;

void initState() {
  super.initState();
  _controller = AnimationController(vsync: this, duration: 300.ms);
}

void dispose() {
  _controller.dispose();
  super.dispose();
}
```

## Common Anti-Patterns

### 1. Rebuilding Entire Widget Tree

```dart
// BAD: setState at top level rebuilds everything
class _HomeState extends State<Home> {
  int _counter = 0;
  void _increment() => setState(() => _counter++);

  Widget build(BuildContext context) {
    return Column(
      children: [
        ExpensiveHeader(),        // Rebuilds unnecessarily
        Text('$_counter'),        // Only this needs updating
        ExpensiveFooter(),        // Rebuilds unnecessarily
      ],
    );
  }
}

// GOOD: Isolate changing state to smallest widget
class _CounterDisplay extends StatefulWidget { ... }
```

### 2. Expensive Operations in build()

```dart
// BAD: Runs every frame
Widget build(BuildContext context) {
  final sorted = List.from(items)..sort(); // O(n log n) per build!
  return ListView(children: sorted.map(ItemWidget.new).toList());
}

// GOOD: Compute outside build
void _updateItems() {
  _sortedItems = List.from(items)..sort();
  setState(() {});
}
```

### 3. Unbounded Image Cache

```dart
// Set reasonable cache limits
PaintingBinding.instance.imageCache.maximumSize = 100;
PaintingBinding.instance.imageCache.maximumSizeBytes = 50 << 20; // 50MB
```

### 4. Opacity Widget for Hiding

```dart
// BAD: Still lays out and paints the widget
Opacity(opacity: 0, child: ExpensiveWidget())

// GOOD: Removes from tree entirely
if (isVisible) ExpensiveWidget(),

// Or use Visibility widget
Visibility(visible: isVisible, child: ExpensiveWidget())
```

## Optimization Checklist

- [ ] `const` constructors used on all eligible widgets
- [ ] `ListView.builder` for lists >20 items
- [ ] Images decoded at display size (cacheWidth/cacheHeight)
- [ ] Heavy computation off UI thread (compute/isolate)
- [ ] All controllers and subscriptions disposed
- [ ] RepaintBoundary on independent animations
- [ ] No expensive operations in build()
- [ ] No Opacity(0) for hiding widgets
- [ ] App profiled in `--profile` mode (not debug)
- [ ] Memory stable over repeated navigation

## Quick Diagnosis

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Scroll jank | ListView without builder | Use ListView.builder |
| First-frame lag | Expensive build() | Move computation outside |
| Growing memory | Undisposed listeners/controllers | Dispose in dispose() |
| Slow images | Full-resolution decode | Use cacheWidth/cacheHeight |
| Choppy animations | Rebuilds crossing RepaintBoundary | Add RepaintBoundary |
| Slow startup | Too much init work | Lazy-load, defer work |
