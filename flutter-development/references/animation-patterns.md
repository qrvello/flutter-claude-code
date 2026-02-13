# Flutter Animation Patterns - Quick Reference

Battle-tested animation patterns for common transitions and effects.

## Table of Contents

- [Implicit Animations](#implicit-animations-easiest)
- [Explicit Animations](#explicit-animations-more-control)
- [Page Transitions](#page-transitions)
- [List Animations](#list-animations)
- [Button Animations](#button-animations)
- [Loading Animations](#loading-animations)
- [Hero Animation](#hero-animation)
- [Usage Tips](#usage-tips)

## Implicit Animations (Easiest)

### AnimatedContainer

```dart
AnimatedContainer(
  duration: Duration(milliseconds: 300),
  curve: Curves.easeInOut,
  width: _expanded ? 200 : 100,
  height: _expanded ? 200 : 100,
  decoration: BoxDecoration(
    color: _expanded ? Colors.blue : Colors.red,
    borderRadius: BorderRadius.circular(_expanded ? 50 : 10),
  ),
  child: Center(child: Text('Tap me')),
)
```

### AnimatedOpacity (Fade)

```dart
AnimatedOpacity(
  opacity: _visible ? 1.0 : 0.0,
  duration: Duration(milliseconds: 500),
  child: YourWidget(),
)
```

### AnimatedPositioned (Slide in Stack)

```dart
Stack(
  children: [
    AnimatedPositioned(
      duration: Duration(milliseconds: 300),
      left: _active ? 0 : 100,
      top: _active ? 0 : 50,
      child: YourWidget(),
    ),
  ],
)
```

## Explicit Animations (More Control)

### Fade In

```dart
class FadeIn extends StatefulWidget {
  final Widget child;
  final Duration duration;
  const FadeIn({required this.child, this.duration = const Duration(milliseconds: 500)});

  @override
  _FadeInState createState() => _FadeInState();
}

class _FadeInState extends State<FadeIn> with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _animation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(duration: widget.duration, vsync: this);
    _animation = Tween<double>(begin: 0.0, end: 1.0).animate(
      CurvedAnimation(parent: _controller, curve: Curves.easeIn),
    );
    _controller.forward();
  }

  @override
  void dispose() { _controller.dispose(); super.dispose(); }

  @override
  Widget build(BuildContext context) => FadeTransition(opacity: _animation, child: widget.child);
}
```

### Slide In

```dart
class SlideIn extends StatefulWidget {
  final Widget child;
  final Offset begin;
  final Duration duration;
  const SlideIn({required this.child, this.begin = const Offset(1, 0), this.duration = const Duration(milliseconds: 300)});

  @override
  _SlideInState createState() => _SlideInState();
}

class _SlideInState extends State<SlideIn> with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<Offset> _animation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(duration: widget.duration, vsync: this);
    _animation = Tween<Offset>(begin: widget.begin, end: Offset.zero)
        .animate(CurvedAnimation(parent: _controller, curve: Curves.easeOut));
    _controller.forward();
  }

  @override
  void dispose() { _controller.dispose(); super.dispose(); }

  @override
  Widget build(BuildContext context) => SlideTransition(position: _animation, child: widget.child);
}
```

## Page Transitions

### Slide Route

```dart
class SlideRoute extends PageRouteBuilder {
  final Widget page;
  SlideRoute({required this.page}) : super(
    pageBuilder: (context, animation, secondaryAnimation) => page,
    transitionsBuilder: (context, animation, secondaryAnimation, child) {
      var tween = Tween(begin: Offset(1.0, 0.0), end: Offset.zero)
          .chain(CurveTween(curve: Curves.easeInOut));
      return SlideTransition(position: animation.drive(tween), child: child);
    },
  );
}

// Usage: Navigator.push(context, SlideRoute(page: DetailsPage()));
```

### Fade Route

```dart
class FadeRoute extends PageRouteBuilder {
  final Widget page;
  FadeRoute({required this.page}) : super(
    pageBuilder: (context, animation, secondaryAnimation) => page,
    transitionsBuilder: (context, animation, secondaryAnimation, child) {
      return FadeTransition(opacity: animation, child: child);
    },
  );
}
```

## List Animations

### Animated List Item (Staggered)

```dart
class AnimatedListItem extends StatefulWidget {
  final Widget child;
  final int index;
  const AnimatedListItem({required this.child, required this.index});

  @override
  _AnimatedListItemState createState() => _AnimatedListItemState();
}

class _AnimatedListItemState extends State<AnimatedListItem> with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<Offset> _slideAnimation;
  late Animation<double> _fadeAnimation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(duration: Duration(milliseconds: 500), vsync: this);
    Future.delayed(Duration(milliseconds: widget.index * 100), () {
      if (mounted) _controller.forward();
    });
    _slideAnimation = Tween<Offset>(begin: Offset(0, 0.5), end: Offset.zero)
        .animate(CurvedAnimation(parent: _controller, curve: Curves.easeOut));
    _fadeAnimation = Tween<double>(begin: 0.0, end: 1.0)
        .animate(CurvedAnimation(parent: _controller, curve: Curves.easeIn));
  }

  @override
  void dispose() { _controller.dispose(); super.dispose(); }

  @override
  Widget build(BuildContext context) {
    return SlideTransition(
      position: _slideAnimation,
      child: FadeTransition(opacity: _fadeAnimation, child: widget.child),
    );
  }
}
```

## Button Animations

### Animated Press Effect

```dart
class AnimatedButton extends StatefulWidget {
  final VoidCallback onPressed;
  final Widget child;
  const AnimatedButton({required this.onPressed, required this.child});

  @override
  _AnimatedButtonState createState() => _AnimatedButtonState();
}

class _AnimatedButtonState extends State<AnimatedButton> with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _scaleAnimation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(duration: Duration(milliseconds: 100), vsync: this);
    _scaleAnimation = Tween<double>(begin: 1.0, end: 0.95)
        .animate(CurvedAnimation(parent: _controller, curve: Curves.easeInOut));
  }

  @override
  void dispose() { _controller.dispose(); super.dispose(); }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTapDown: (_) => _controller.forward(),
      onTapUp: (_) { _controller.reverse(); widget.onPressed(); },
      onTapCancel: () => _controller.reverse(),
      child: ScaleTransition(scale: _scaleAnimation, child: widget.child),
    );
  }
}
```

## Loading Animations

### Shimmer Loading

```dart
class ShimmerLoading extends StatefulWidget {
  final Widget child;
  const ShimmerLoading({required this.child});

  @override
  _ShimmerLoadingState createState() => _ShimmerLoadingState();
}

class _ShimmerLoadingState extends State<ShimmerLoading> with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _animation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(duration: Duration(milliseconds: 1500), vsync: this)..repeat();
    _animation = Tween<double>(begin: -2, end: 2).animate(_controller);
  }

  @override
  void dispose() { _controller.dispose(); super.dispose(); }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _animation,
      builder: (context, child) {
        return ShaderMask(
          shaderCallback: (bounds) => LinearGradient(
            stops: [_animation.value - 0.3, _animation.value, _animation.value + 0.3],
            colors: [Colors.grey[300]!, Colors.grey[100]!, Colors.grey[300]!],
          ).createShader(bounds),
          child: widget.child,
        );
      },
    );
  }
}
```

## Hero Animation

```dart
// Source screen
Hero(
  tag: 'product-${product.id}',
  child: Image.network(product.imageUrl),
)

// Destination screen
Hero(
  tag: 'product-${product.id}',
  child: Image.network(product.imageUrl, width: double.infinity, height: 300, fit: BoxFit.cover),
)
```

## Usage Tips

1. **Performance**: Use `RepaintBoundary` for complex animations
2. **Dispose**: Always dispose controllers in `dispose()`
3. **vsync**: Use `SingleTickerProviderStateMixin` or `TickerProviderStateMixin`
4. **Curves**: Experiment with different curves (easeIn, easeOut, elasticOut, bounceOut)
5. **Duration guidelines**:
   - Micro interactions: 100-200ms
   - Standard transitions: 300-500ms
   - Complex animations: 500-1000ms
