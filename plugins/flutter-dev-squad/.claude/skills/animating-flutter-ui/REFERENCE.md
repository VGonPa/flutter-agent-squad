# Animation Reference

Quick-reference for Flutter animation APIs. For decision guidance and patterns, see [SKILL.md](SKILL.md).

## Implicit Animation Widgets

| Widget | Animates | Example Use |
|--------|----------|-------------|
| `AnimatedContainer` | Size, color, padding, margin, decoration | Expandable cards, state-driven styling |
| `AnimatedOpacity` | Opacity (0.0 → 1.0) | Fade in/out |
| `AnimatedPositioned` | Position inside a Stack | Sliding panels, repositioning |
| `AnimatedPadding` | Padding | Content area expansion |
| `AnimatedAlign` | Alignment | Widget repositioning within parent |
| `AnimatedDefaultTextStyle` | Text style (size, weight, color) | Emphasis changes |
| `AnimatedCrossFade` | Cross-fade between two fixed children | Show/hide with `crossFadeState` |
| `AnimatedSize` | Size with clipping | Collapsible sections |

All accept `duration` and optional `curve` parameters.

## Curves Reference

### Common Curves

| Curve | Use Case | Character |
|-------|----------|-----------|
| `Curves.easeInOut` | General purpose | Smooth start and end |
| `Curves.easeOut` | Entering elements | Fast start, gentle landing |
| `Curves.easeIn` | Exiting elements | Gentle start, fast exit |
| `Curves.easeInOutCubicEmphasized` | Material 3 standard | M3 recommended default |
| `Curves.elasticOut` | Bouncy/playful entrance | Spring overshoot |
| `Curves.bounceOut` | Bouncing to rest | Gravity-like bounce |
| `Curves.linear` | Constant speed | Progress bars, loading |

### Material 3 Motion Tokens

| Motion Type | Curve | Duration | Use |
|-------------|-------|----------|-----|
| Standard | `Curves.easeInOutCubicEmphasized` | 500ms | Page transitions, expanding panels |
| Entering | `Curves.easeOutCubic` | 400ms | Elements appearing on screen |
| Exiting | `Curves.easeInCubic` | 200ms | Elements leaving screen |
| Micro interaction | `Curves.easeInOut` | 100-200ms | Tap feedback, toggles |

## Duration Guidelines

| Animation Type | Duration | Notes |
|---------------|----------|-------|
| Micro interaction (tap feedback) | 100-200ms | Should feel instant |
| Simple transition (fade, scale) | 200-300ms | Noticeable but quick |
| Standard transition (page, slide) | 300-500ms | M3 standard is 500ms |
| Complex animation (staggered) | 500-1000ms | Cap individual items |
| Material 3 standard | 500ms | Default for page transitions |

## Tween Types

```dart
Tween<double>(begin: 0.0, end: 1.0)                        // Opacity, scale, rotation
Tween<Offset>(begin: Offset(1, 0), end: Offset.zero)       // Slide position
ColorTween(begin: Colors.red, end: Colors.blue)             // Color transitions
IntTween(begin: 0, end: 100)                                // Counter animations
SizeTween(begin: Size(100, 100), end: Size(200, 200))       // Size transitions
```

## Transition Widgets (for Explicit Animations)

Each pairs with an `AnimationController` + appropriate `Tween`:

| Effect | Tween Type | Transition Widget |
|--------|-----------|-------------------|
| Fade | `Tween<double>` | `FadeTransition(opacity: animation)` |
| Slide | `Tween<Offset>` | `SlideTransition(position: animation)` |
| Scale | `Tween<double>` | `ScaleTransition(scale: animation)` |
| Rotation | `Tween<double>` | `RotationTransition(turns: animation)` |
| Size | `Tween<double>` | `SizeTransition(sizeFactor: animation)` |

## Controller Methods

```dart
_controller.forward();                                // Play forward once
_controller.reverse();                                // Play backward once
_controller.repeat();                                 // Loop forever
_controller.repeat(reverse: true);                    // Pulse (forward → reverse → forward...)
_controller.forward().then((_) => _controller.reverse()); // Play once then reverse
_controller.animateTo(0.5);                           // Animate to specific value
_controller.value = 0.5;                              // Jump to value (no animation)
_controller.reset();                                  // Jump to begin value
_controller.stop();                                   // Stop at current value
```

## Staggered List Implementation

```dart
class AnimatedListItem extends StatefulWidget {
  final Widget child;
  final int index;
  const AnimatedListItem({super.key, required this.child, required this.index});
  @override
  State<AnimatedListItem> createState() => _AnimatedListItemState();
}

class _AnimatedListItemState extends State<AnimatedListItem>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;
  late final Animation<Offset> _slide;
  late final Animation<double> _fade;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      duration: const Duration(milliseconds: 500), vsync: this);
    _slide = Tween<Offset>(begin: const Offset(0, 0.3), end: Offset.zero)
        .animate(CurvedAnimation(parent: _controller, curve: Curves.easeOut));
    _fade = Tween<double>(begin: 0.0, end: 1.0)
        .animate(CurvedAnimation(parent: _controller, curve: Curves.easeIn));

    // Cap delay for large lists: max 10 items staggered
    final delay = Duration(milliseconds: (widget.index.clamp(0, 10)) * 100);
    Future.delayed(delay, () {
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
final items = <String>[];

// Widget
AnimatedList(
  key: _listKey,
  initialItemCount: items.length,
  itemBuilder: (context, index, animation) =>
      SizeTransition(
        sizeFactor: animation,
        child: ListTile(title: Text(items[index])),
      ),
)

// Insert with animation
void addItem(String item) {
  items.insert(0, item);
  _listKey.currentState?.insertItem(0);
}

// Remove with animation
void removeItem(int index) {
  final removed = items.removeAt(index);
  _listKey.currentState?.removeItem(index,
    (context, animation) => SizeTransition(
      sizeFactor: animation,
      child: ListTile(title: Text(removed)),
    ),
  );
}
```

## Third-Party Library Initialization

### Lottie

```dart
// dependencies: lottie: ^3.0.0

// Simple playback (auto-plays)
Lottie.asset('assets/animations/loading.json')

// With controller for manual control
late final AnimationController _controller;

Lottie.asset(
  'assets/animations/success.json',
  controller: _controller,
  onLoaded: (composition) {
    _controller.duration = composition.duration;
    _controller.forward();
  },
)
```

### Rive

```dart
// dependencies: rive: ^0.13.0

// Simple playback
RiveAnimation.asset('assets/animations/button.riv')

// With state machine for interactivity
late SMIBool _isPressed;

RiveAnimation.asset(
  'assets/animations/button.riv',
  onInit: (artboard) {
    final ctrl = StateMachineController.fromArtboard(artboard, 'State Machine');
    artboard.addController(ctrl!);
    _isPressed = ctrl.findInput<bool>('isPressed') as SMIBool;
  },
)

// Trigger state change
_isPressed.value = true;
```
