---
name: building-flutter-widgets
description: Guides Flutter widget composition with Material Design 3, responsive layouts, theming, accessibility, and extraction patterns. Use when building UI components, designing responsive screens, setting up ThemeExtension, or deciding when to extract widgets.
---

# Building Flutter Widgets

Best practices for composing Flutter widgets with Material Design 3, responsive layouts, accessibility, and clean extraction patterns.

## When to Use This Skill

- Building new UI components or screens
- Implementing Material Design 3 theming
- Making layouts responsive across screen sizes
- Adding accessibility (Semantics) to widgets
- Deciding when and how to extract widgets
- Setting up ThemeExtension for custom tokens

## Widget Composition Principles

### Prefer Composition Over Inheritance

```dart
// BAD: Inheriting from Material widgets
class MyButton extends ElevatedButton { ... }

// GOOD: Composing with Material widgets
class PrimaryButton extends StatelessWidget {
  const PrimaryButton({super.key, required this.onPressed, required this.label});
  final VoidCallback? onPressed;
  final String label;

  @override
  Widget build(BuildContext context) {
    return FilledButton(
      onPressed: onPressed,
      child: Text(label),
    );
  }
}
```

### Const Constructors

Always use `const` constructors when possible — enables Flutter to skip rebuilds:

```dart
// GOOD: const constructor + const usage
class StatusBadge extends StatelessWidget {
  const StatusBadge({super.key, required this.label, this.color});
  final String label;
  final Color? color;

  @override
  Widget build(BuildContext context) {
    final colorScheme = Theme.of(context).colorScheme;
    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
      decoration: BoxDecoration(
        color: color ?? colorScheme.primaryContainer,
        borderRadius: BorderRadius.circular(12),
      ),
      child: Text(label, style: Theme.of(context).textTheme.labelSmall),
    );
  }
}

// Usage — const when all args are compile-time constants
const StatusBadge(label: 'Active')
```

**When `const` is NOT possible:**
- Constructor takes runtime values (`DateTime.now()`, API data)
- Uses non-const default values

### StatelessWidget vs StatefulWidget

| Use StatelessWidget when... | Use StatefulWidget when... |
|-----------------------------|---------------------------|
| UI depends only on constructor args | Widget has internal mutable state |
| State comes from external source (Riverpod) | AnimationController, TextEditingController |
| Pure display component | Focus nodes, scroll controllers |
| No lifecycle methods needed | initState/dispose cleanup required |

## Material Design 3

### Core M3 Widgets

```dart
// Navigation
NavigationBar         // Bottom nav (replaces BottomNavigationBar)
NavigationRail        // Side nav for tablets
NavigationDrawer      // Slide-out drawer

// Buttons (by emphasis, high → low)
FilledButton          // Primary action
FilledButton.tonal    // Secondary action
OutlinedButton        // Medium emphasis
TextButton            // Low emphasis
IconButton            // Icon-only action
FloatingActionButton  // Primary screen action

// Containers
Card                  // Surface container
Dialog                // Modal overlay
BottomSheet           // Bottom panel
SearchBar / SearchAnchor // Search interface

// Inputs
TextField(decoration: InputDecoration(...))
DropdownMenu          // Replaces DropdownButton
SegmentedButton       // Toggle group
FilterChip / ChoiceChip / InputChip
```

### Using ColorScheme (Never Hardcode Colors)

```dart
// BAD: Hardcoded colors
Container(color: Color(0xFF6200EE))
Text('Error', style: TextStyle(color: Colors.red))

// GOOD: ColorScheme tokens
final cs = Theme.of(context).colorScheme;
Container(color: cs.primaryContainer)
Text('Error', style: TextStyle(color: cs.error))
```

**Key ColorScheme roles:**
- `primary` / `onPrimary` — Brand color + text on it
- `secondary` / `onSecondary` — Accent elements
- `surface` / `onSurface` — Card/sheet backgrounds
- `error` / `onError` — Error states
- `primaryContainer` / `onPrimaryContainer` — Subtle brand backgrounds

### Typography with TextTheme

```dart
final tt = Theme.of(context).textTheme;

Text('Headline', style: tt.headlineMedium)
Text('Body copy', style: tt.bodyLarge)
Text('Caption', style: tt.labelSmall)
```

**Scale:** `displayLarge > headlineLarge > titleLarge > bodyLarge > labelLarge`
Each has Large / Medium / Small variants.

## Theming & ThemeExtension

### App Theme Setup

```dart
class AppTheme {
  static ThemeData light() {
    final colorScheme = ColorScheme.fromSeed(
      seedColor: const Color(0xFF1A73E8),
      brightness: Brightness.light,
    );
    return ThemeData(
      useMaterial3: true,
      colorScheme: colorScheme,
      textTheme: _textTheme,
      extensions: [AppSpacing.regular, AppSizes.regular],
    );
  }

  static ThemeData dark() {
    final colorScheme = ColorScheme.fromSeed(
      seedColor: const Color(0xFF1A73E8),
      brightness: Brightness.dark,
    );
    return ThemeData(
      useMaterial3: true,
      colorScheme: colorScheme,
      textTheme: _textTheme,
      extensions: [AppSpacing.regular, AppSizes.regular],
    );
  }
}
```

### Custom ThemeExtension

```dart
class AppSpacing extends ThemeExtension<AppSpacing> {
  final double xs, sm, md, lg, xl;

  const AppSpacing({
    required this.xs, required this.sm, required this.md,
    required this.lg, required this.xl,
  });

  static const regular = AppSpacing(xs: 4, sm: 8, md: 16, lg: 24, xl: 32);

  @override
  AppSpacing copyWith({double? xs, double? sm, double? md, double? lg, double? xl}) {
    return AppSpacing(
      xs: xs ?? this.xs, sm: sm ?? this.sm, md: md ?? this.md,
      lg: lg ?? this.lg, xl: xl ?? this.xl,
    );
  }

  @override
  AppSpacing lerp(covariant AppSpacing? other, double t) {
    if (other == null) return this;
    return AppSpacing(
      xs: lerpDouble(xs, other.xs, t)!, sm: lerpDouble(sm, other.sm, t)!,
      md: lerpDouble(md, other.md, t)!, lg: lerpDouble(lg, other.lg, t)!,
      xl: lerpDouble(xl, other.xl, t)!,
    );
  }
}

// Usage
final spacing = Theme.of(context).extension<AppSpacing>()!;
Padding(padding: EdgeInsets.all(spacing.md), child: ...)
```

## Responsive Layouts

### LayoutBuilder + Breakpoints

```dart
class ResponsiveLayout extends StatelessWidget {
  const ResponsiveLayout({super.key, required this.mobile, this.tablet, this.desktop});
  final Widget mobile;
  final Widget? tablet;
  final Widget? desktop;

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(builder: (context, constraints) {
      if (constraints.maxWidth >= 1200) return desktop ?? tablet ?? mobile;
      if (constraints.maxWidth >= 600) return tablet ?? mobile;
      return mobile;
    });
  }
}
```

### MediaQuery for Screen-Level Decisions

```dart
// Screen dimensions
final size = MediaQuery.sizeOf(context);
final padding = MediaQuery.paddingOf(context);

// Prefer specific MediaQuery methods (avoids unnecessary rebuilds)
// GOOD:
MediaQuery.sizeOf(context)
MediaQuery.paddingOf(context)

// BAD (rebuilds on ANY MediaQuery change):
MediaQuery.of(context).size
```

### Adaptive Patterns

```dart
// Grid that adapts column count
GridView.builder(
  gridDelegate: SliverGridDelegateWithMaxCrossAxisExtent(
    maxCrossAxisExtent: 300,  // Flutter calculates column count
    mainAxisSpacing: 16,
    crossAxisSpacing: 16,
  ),
  itemBuilder: (context, index) => ItemCard(item: items[index]),
)

// Wrap for flexible row layouts
Wrap(
  spacing: 8,
  runSpacing: 8,
  children: tags.map((tag) => FilterChip(label: Text(tag), ...)).toList(),
)
```

## Accessibility

### Semantics

```dart
// Label non-text elements
Semantics(
  label: 'User profile photo',
  image: true,
  child: CircleAvatar(backgroundImage: NetworkImage(url)),
)

// Merge semantics for grouped content
MergeSemantics(
  child: ListTile(
    leading: Icon(Icons.email),
    title: Text(email),
    subtitle: Text('Email address'),
  ),
)

// Exclude decorative elements
ExcludeSemantics(child: DecorativeBackground())

// Announce state changes
SemanticsService.announce('Item added to cart', TextDirection.ltr);
```

### Key Accessibility Rules

1. **Touch targets:** Minimum 48x48 logical pixels
2. **Contrast:** 4.5:1 for normal text, 3:1 for large text — M3 ColorScheme handles this
3. **Labels:** All interactive elements need semantic labels
4. **Focus order:** Logical tab/swipe order (use `FocusTraversalGroup`)
5. **Text scaling:** Use `textScaler` aware layouts, avoid fixed heights for text containers

```dart
// BAD: Fixed height breaks with large text
SizedBox(height: 20, child: Text('Label'))

// GOOD: Flexible height
Text('Label', style: tt.bodyMedium)
```

## Widget Extraction Patterns

### When to Extract

| Signal | Action |
|--------|--------|
| build() > 80 lines | Extract sub-widgets |
| Same widget tree in 2+ places | Extract to shared widget |
| Widget has its own state/logic | Extract to StatefulWidget |
| Distinct visual section | Extract for readability |
| Needs independent testing | Extract for testability |

### Extraction Strategies

**Strategy 1: Private Widget in Same File**
```dart
class ProductPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Column(children: [_Header(), _ProductList(), _Footer()]);
  }
}

class _Header extends StatelessWidget { ... }
class _ProductList extends StatelessWidget { ... }
class _Footer extends StatelessWidget { ... }
```

**Strategy 2: Public Widget in Separate File**
```dart
// widgets/product_card.dart
class ProductCard extends StatelessWidget {
  const ProductCard({super.key, required this.product, this.onTap});
  final Product product;
  final VoidCallback? onTap;
  // ...
}
```

**Strategy 3: Builder Callback (Avoid Prop Drilling)**
```dart
class DataList<T> extends StatelessWidget {
  const DataList({super.key, required this.items, required this.itemBuilder});
  final List<T> items;
  final Widget Function(BuildContext, T) itemBuilder;

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: items.length,
      itemBuilder: (context, index) => itemBuilder(context, items[index]),
    );
  }
}
```

### File Organization

```
features/products/presentation/
├── pages/
│   └── product_list_page.dart     # Screen-level widget (Scaffold)
└── widgets/
    ├── product_card.dart           # Reusable within feature
    ├── product_filters.dart        # Feature-specific
    └── product_empty_state.dart    # Feature-specific

shared/widgets/
├── buttons/
│   └── primary_button.dart         # App-wide reusable
├── feedback/
│   └── empty_state.dart            # Generic empty state
└── layout/
    └── responsive_layout.dart      # Layout utility
```

## Common Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Hardcoded colors/sizes | Doesn't respect theme | Use ColorScheme, TextTheme, ThemeExtension |
| `MediaQuery.of(context)` | Rebuilds on any change | Use `MediaQuery.sizeOf(context)` |
| Deep widget nesting (>5 levels) | Unreadable | Extract sub-widgets |
| Business logic in build() | Mixing concerns | Move to controller |
| Missing const constructors | Missed optimization | Add `const` to constructors and usage sites |
| Text in fixed-height containers | Breaks with text scaling | Use flexible layouts |

## Quick Checklist (New Widget)

- [ ] `const` constructor if possible?
- [ ] Uses `Theme.of(context)` for colors/typography?
- [ ] Responsive to different screen sizes?
- [ ] Semantic labels on interactive/image elements?
- [ ] Touch targets >= 48x48?
- [ ] build() method under 80 lines?
- [ ] No hardcoded strings (use localization)?
- [ ] Key provided for list items?
