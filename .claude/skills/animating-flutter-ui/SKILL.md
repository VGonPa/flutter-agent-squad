---
name: animating-flutter-ui
description: Implements Flutter animations and transitions. Use when adding implicit/explicit animations, page transitions, Hero animations, staggered lists, or integrating Rive/Lottie. Covers AnimationController, Curves, Tween, TweenAnimationBuilder, and Material 3 motion patterns.
---

# Animating Flutter UI

Animation patterns, transitions, and motion design for Flutter applications.

## When to Use This Skill

- Adding animations to widgets (fade, slide, scale, rotate)
- Choosing between implicit and explicit animations
- Implementing page transitions
- Creating staggered list animations
- Adding Hero transitions between screens
- Integrating Rive or Lottie animations
- Following Material 3 motion guidelines

## Implicit vs Explicit: Decision Guide

```
"I need to animate..."

A property change (size, color, position)?
  -> Implicit animation (AnimatedContainer, AnimatedOpacity)
  -> Simplest approach, no controller needed

Continuous/repeating motion?
  -> Explicit animation (AnimationController)
  -> Full control over timing and lifecycle

A transition between two child widgets?
  -> AnimatedSwitcher
  -> Automatic cross-fade/scale between old and new child

A one-shot animation on build?
  -> TweenAnimationBuilder
  -> No controller needed, plays once
```

## Implicit Animations (Simplest)

### AnimatedContainer

```dart
class AnimatedBox extends StatefulWidget {
  @override
  State<AnimatedBox> createState() => _AnimatedBoxState();
}

class _AnimatedBoxState extends State<AnimatedBox> {
  bool _expanded = false;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () => setState(() => _expanded = !_expanded),
      child: AnimatedContainer(
        duration: const Duration(milliseconds: 300),
        curve: Curves.easeInOut,
        width: _expanded ? 200 : 100,
        height: _expanded ? 200 : 100,
        decoration: BoxDecoration(
          color: _expanded ? Colors.blue : Colors.red,
          borderRadius: BorderRadius.circular(_expanded ? 50 : 10),
        ),
      ),
    );
  }
}
```

### AnimatedOpacity (Fade)

```dart
AnimatedOpacity(
  opacity: _visible ? 1.0 : 0.0,
  duration: const Duration(milliseconds: 500),
  child: YourWidget(),
)
```

### AnimatedPositioned (Slide in Stack)

```dart
Stack(
  children: [
    AnimatedPositioned(
      duration: const Duration(milliseconds: 300),
      left: _active ? 0 : 100,
      top: _active ? 0 : 50,
      child: YourWidget(),
    ),
  ],
)
```

### Other Implicit Widgets

| Widget | Animates |
|--------|----------|
| `AnimatedContainer` | Size, color, padding, margin, decoration |
| `AnimatedOpacity` | Opacity (fade in/out) |
| `AnimatedPositioned` | Position in Stack |
| `AnimatedPadding` | Padding |
| `AnimatedAlign` | Alignment |
| `AnimatedDefaultTextStyle` | Text style |
| `AnimatedCrossFade` | Cross-fade between two children |
| `AnimatedSize` | Size with clipping |

## Explicit Animations (Full Control)

### AnimationController Basics

```dart
class FadeIn extends StatefulWidget {
  final Widget child;
  final Duration duration;

  const FadeIn({
    required this.child,
    this.duration = const Duration(milliseconds: 500),
  });

  @override
  State<FadeIn> createState() => _FadeInState();
}

class _FadeInState extends State<FadeIn>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _animation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      duration: widget.duration,
      vsync: this,
    );
    _animation = CurvedAnimation(
      parent: _controller,
      curve: Curves.easeIn,
    );
    _controller.forward();
  }

  @override
  void dispose() {
    _controller.dispose(); // ALWAYS dispose!
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return FadeTransition(
      opacity: _animation,
      child: widget.child,
    );
  }
}
```

### Other Transition Widgets

Same AnimationController pattern as FadeIn above, just change the Tween and transition widget:

```dart
// Slide: Tween<Offset> + SlideTransition
Tween<Offset>(begin: const Offset(1, 0), end: Offset.zero)
SlideTransition(position: _animation, child: widget.child)

// Scale: Tween<double> + ScaleTransition
Tween<double>(begin: 0.0, end: 1.0)  // with Curves.elasticOut for bounce
ScaleTransition(scale: _animation, child: widget.child)

// Rotation: Tween<double> + RotationTransition
Tween<double>(begin: 0.0, end: 1.0)  // 1.0 = full rotation
RotationTransition(turns: _animation, child: widget.child)
```

### Controller Methods

```dart
_controller.forward();                              // Play once
_controller.reverse();                              // Play backward
_controller.repeat();                               // Loop forever
_controller.repeat(reverse: true);                  // Pulse/breathe
_controller.forward().then((_) => _controller.reverse()); // Play then reverse
```

## Curves Reference

### Common Curves

| Curve | Use Case |
|-------|----------|
| `Curves.easeInOut` | General purpose |
| `Curves.easeOut` | Entering elements |
| `Curves.easeIn` | Exiting elements |
| `Curves.easeInOutCubicEmphasized` | Material 3 standard |
| `Curves.elasticOut` | Bouncy/playful entrance |
| `Curves.bounceOut` | Bouncing to rest |
| `Curves.linear` | Constant speed (progress bars) |

### Material 3 Motion Tokens

| Motion Type | Curve | Duration |
|-------------|-------|----------|
| Standard | `Curves.easeInOutCubicEmphasized` | 500ms |
| Entering | `Curves.easeOutCubic` | 400ms |
| Exiting | `Curves.easeInCubic` | 200ms |
| Micro interaction | `Curves.easeInOut` | 100-200ms |

## Tween Types

```dart
Tween<double>(begin: 0.0, end: 1.0)             // Opacity, scale, rotation
Tween<Offset>(begin: Offset(1, 0), end: Offset.zero)  // Slide position
ColorTween(begin: Colors.red, end: Colors.blue)  // Color transitions
IntTween(begin: 0, end: 100)                     // Counter animations
```

## TweenAnimationBuilder (No Controller)

```dart
TweenAnimationBuilder<double>(
  tween: Tween(begin: 0.0, end: _isExpanded ? 1.0 : 0.0),
  duration: const Duration(milliseconds: 300),
  builder: (context, value, child) {
    return Transform.scale(
      scale: 0.8 + (0.2 * value),
      child: Opacity(opacity: value, child: child),
    );
  },
  child: const Card(child: Text('Animated content')),
)
```

## AnimatedSwitcher

```dart
AnimatedSwitcher(
  duration: const Duration(milliseconds: 300),
  transitionBuilder: (child, animation) {
    return FadeTransition(
      opacity: animation,
      child: ScaleTransition(scale: animation, child: child),
    );
  },
  // Key is ESSENTIAL for detecting changes
  child: _showCheck
      ? const Icon(Icons.check, key: ValueKey('check'))
      : const Icon(Icons.close, key: ValueKey('close')),
)
```

## Page Transitions

### Custom Page Transitions

```dart
// Slide from right
class SlideRoute extends PageRouteBuilder {
  final Widget page;
  SlideRoute({required this.page}) : super(
    pageBuilder: (_, __, ___) => page,
    transitionsBuilder: (_, animation, __, child) => SlideTransition(
      position: Tween(begin: const Offset(1.0, 0.0), end: Offset.zero)
          .chain(CurveTween(curve: Curves.easeInOut)).animate(animation),
      child: child,
    ),
  );
}

// Fade
transitionsBuilder: (_, animation, __, child) =>
    FadeTransition(opacity: animation, child: child),

// Material 3 (fade + subtle slide up)
transitionDuration: const Duration(milliseconds: 500),
reverseTransitionDuration: const Duration(milliseconds: 200),
transitionsBuilder: (_, animation, __, child) => FadeTransition(
  opacity: CurvedAnimation(parent: animation, curve: Curves.easeOutCubic),
  child: SlideTransition(
    position: Tween(begin: const Offset(0, 0.05), end: Offset.zero)
        .animate(CurvedAnimation(parent: animation,
            curve: Curves.easeInOutCubicEmphasized)),
    child: child,
  ),
),

// Usage
Navigator.push(context, SlideRoute(page: DetailsPage()));
```

## Hero Animations

```dart
// Source screen
GestureDetector(
  onTap: () => Navigator.push(
    context,
    MaterialPageRoute(builder: (_) => DetailScreen(product)),
  ),
  child: Hero(
    tag: 'product-${product.id}',
    child: Image.network(product.imageUrl),
  ),
)

// Destination screen
Hero(
  tag: 'product-${product.id}',
  child: Image.network(
    product.imageUrl,
    width: double.infinity,
    height: 300,
    fit: BoxFit.cover,
  ),
)
```

**Rules:**
- `tag` must be unique and identical on both screens
- Both Hero children should be the same widget type for smooth morphing
- Works automatically with MaterialPageRoute

## Staggered List Animation

```dart
// Stagger items by delaying each based on index
class AnimatedListItem extends StatefulWidget {
  final Widget child;
  final int index;
  const AnimatedListItem({required this.child, required this.index});
  @override
  State<AnimatedListItem> createState() => _AnimatedListItemState();
}

class _AnimatedListItemState extends State<AnimatedListItem>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<Offset> _slide;
  late Animation<double> _fade;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      duration: const Duration(milliseconds: 500), vsync: this);
    _slide = Tween<Offset>(begin: const Offset(0, 0.5), end: Offset.zero)
        .animate(CurvedAnimation(parent: _controller, curve: Curves.easeOut));
    _fade = Tween<double>(begin: 0.0, end: 1.0)
        .animate(CurvedAnimation(parent: _controller, curve: Curves.easeIn));
    Future.delayed(Duration(milliseconds: widget.index * 100), () {
      if (mounted) _controller.forward();
    });
  }

  @override
  void dispose() { _controller.dispose(); super.dispose(); }

  @override
  Widget build(BuildContext context) {
    return SlideTransition(
      position: _slide,
      child: FadeTransition(opacity: _fade, child: widget.child),
    );
  }
}
```

## AnimatedList (Insert/Remove)

```dart
final _listKey = GlobalKey<AnimatedListState>();

AnimatedList(
  key: _listKey,
  initialItemCount: items.length,
  itemBuilder: (context, index, animation) =>
      SizeTransition(sizeFactor: animation, child: ListTile(title: Text(items[index]))),
)

void _addItem(String item) {
  items.insert(0, item);
  _listKey.currentState?.insertItem(0);
}

void _removeItem(int index) {
  final removed = items.removeAt(index);
  _listKey.currentState?.removeItem(index,
    (context, animation) => SizeTransition(
      sizeFactor: animation, child: ListTile(title: Text(removed))));
}
```

## Third-Party Animation Libraries

### flutter_animate (Declarative Composition)

```dart
// dependencies: flutter_animate: ^4.5.0
import 'package:flutter_animate/flutter_animate.dart';

Text('Hello')
    .animate()
    .fadeIn(duration: 600.ms)
    .slideY(begin: 0.3, end: 0)
    .then(delay: 200.ms)
    .shake();
```

### Lottie (After Effects Animations)

```dart
// dependencies: lottie: ^3.0.0
Lottie.asset('assets/animations/loading.json')

// With controller for play control
Lottie.asset(
  'assets/animations/success.json',
  controller: _controller, // AnimationController
  onLoaded: (composition) {
    _controller.duration = composition.duration;
    _controller.forward();
  },
)
```

### Rive (Interactive Vector Animations)

```dart
// dependencies: rive: ^0.13.0
// Simple playback
RiveAnimation.asset('assets/animations/button.riv')

// With state machine for interactivity
RiveAnimation.asset(
  'assets/animations/button.riv',
  onInit: (artboard) {
    final ctrl = StateMachineController.fromArtboard(artboard, 'State Machine');
    artboard.addController(ctrl!);
    _isPressed = ctrl.findInput<bool>('isPressed') as SMIBool;
  },
)
```

## Accessibility: Reduced Motion

Always respect the user's reduced motion preference:

```dart
@override
Widget build(BuildContext context) {
  final reduceMotion = MediaQuery.disableAnimationsOf(context);

  return AnimatedContainer(
    duration: reduceMotion
        ? Duration.zero         // Instant transition
        : const Duration(milliseconds: 300),
    curve: Curves.easeInOut,
    // ... properties
  );
}

// For explicit animations: skip or shorten
if (!reduceMotion) {
  _controller.forward();
} else {
  _controller.value = 1.0; // Jump to end state
}
```

**Rule:** Never assume all users want motion. `MediaQuery.disableAnimationsOf(context)` checks both platform accessibility settings (iOS Reduce Motion, Android Remove Animations) and any app-level overrides.

## Performance Tips

1. **RepaintBoundary**: Wrap animated widgets to avoid repainting siblings
2. **Dispose controllers**: Always dispose in `dispose()` to prevent memory leaks
3. **vsync**: Always use `SingleTickerProviderStateMixin` or `TickerProviderStateMixin`
4. **Transform over layout**: `Transform.translate` is cheaper than changing `Padding`/`Positioned`
5. **Impeller**: Shader jank eliminated - animations are smooth from first frame
6. **Avoid `pumpAndSettle` in tests** with infinite animations - use `pump()` instead

## Duration Guidelines

| Animation Type | Duration |
|---------------|----------|
| Micro interaction (tap feedback) | 100-200ms |
| Simple transition (fade, scale) | 200-300ms |
| Standard transition (page, slide) | 300-500ms |
| Complex animation (staggered) | 500-1000ms |
| Material 3 standard | 500ms |
