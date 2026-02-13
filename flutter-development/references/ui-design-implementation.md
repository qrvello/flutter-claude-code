# Flutter UI Design-to-Implementation Reference

Consolidated procedural reference for the full design-to-implementation pipeline:
analyzing designs, generating Flutter code, visually comparing results, and
iterating to pixel-perfect fidelity.

---

## Table of Contents

1. [Design Analysis Workflow](#1-design-analysis-workflow)
   - [Six-Step Analysis Process](#six-step-analysis-process)
   - [Widget Selection Decision Tree](#widget-selection-decision-tree)
   - [Layout Planning](#layout-planning)
   - [Design System Mapping](#design-system-mapping)
   - [Responsive Design Patterns](#responsive-design-patterns)
2. [Code Generation](#2-code-generation)
   - [Performance-First Approach](#performance-first-approach)
   - [Styling with Material 3 and BoxDecoration](#styling-with-material-3-and-boxdecoration)
   - [Responsive Layout Implementation](#responsive-layout-implementation)
   - [Animations](#animations)
   - [Accessibility](#accessibility)
3. [Visual Comparison Workflow](#3-visual-comparison-workflow)
   - [Environment Setup](#environment-setup)
   - [The Crop-Zoom-Compare Methodology](#the-crop-zoom-compare-methodology)
   - [Color Sampling with ImageMagick](#color-sampling-with-imagemagick)
   - [Component Analysis Checklist](#component-analysis-checklist)
   - [Comparison Report Format](#comparison-report-format)
4. [Iteration Workflow](#4-iteration-workflow)
   - [Four-Phase Pipeline](#four-phase-pipeline)
   - [Iteration Management Rules](#iteration-management-rules)
   - [Fix Prioritization Scoring](#fix-prioritization-scoring)
   - [Error Handling Strategies](#error-handling-strategies)
   - [Multi-Screen Designs](#multi-screen-designs)

---

## 1. Design Analysis Workflow

### Six-Step Analysis Process

When analyzing any design file (Figma export, screenshot, mockup), follow these
steps in order:

1. **High-Level Structure** -- Identify the page scaffold (AppBar, body, bottom
   nav), scrolling behavior (whole page vs. sections), and major sections.

2. **Section Breakdown** -- Break design into logical sections (header, content,
   footer). Identify the layout pattern for each (Column, Row, Grid, Stack).
   Note spacing and alignment.

3. **Component Identification** -- List every UI component (buttons, cards,
   images, text, icons). Map each to a Flutter widget. Flag repeated patterns
   that should become custom widgets.

4. **Styling Analysis** -- Extract colors into a color scheme. Identify
   typography styles for a text theme. Note border radii, shadows, and
   decorative elements. Map to Material 3 or custom styles.

5. **Responsive Considerations** -- Identify breakpoints. Plan how layouts
   adapt (column-to-row, grid column counts). Determine which elements
   grow/shrink.

6. **Implementation Plan** -- Produce a widget hierarchy tree, layout specs,
   color/typography mappings, custom widget extractions, and responsive behavior
   specs.

### Widget Selection Decision Tree

Core principle: **composition over complexity**. Prefer multiple simple widgets
over one complex custom widget. Prefer built-in widgets over packages. Use
`const` constructors wherever possible.

#### Single-Child Layout Widgets

| Widget       | Use When                                       |
|------------- |------------------------------------------------|
| `Padding`    | Only padding needed (faster than Container)    |
| `Center`     | Centering a child                              |
| `Align`      | Positioning child at a specific alignment      |
| `SizedBox`   | Fixed size, or spacing in Row/Column           |
| `AspectRatio`| Maintaining proportions (images, video)        |
| `Container`  | Need padding + margin + decoration together    |

#### Multi-Child Layout Widgets

| Widget  | Use When                                            |
|---------|-----------------------------------------------------|
| `Row`   | Horizontal arrangement                              |
| `Column`| Vertical arrangement                                |
| `Stack` | Layered/overlapping elements                        |
| `Wrap`  | Flowing layout that wraps to next line (tags, chips)|

#### Scrolling Widgets

| Widget                      | Use When                                    |
|-----------------------------|---------------------------------------------|
| `ListView.builder`          | Large or infinite lists (lazy loads)         |
| `ListView.separated`        | List with dividers between items             |
| `GridView.builder`          | Large grids (lazy loads)                     |
| `SingleChildScrollView`     | Making a non-scrollable widget scrollable    |
| `CustomScrollView` + Slivers| Advanced scroll effects (collapsing headers) |

#### Flexible Space Distribution

```dart
Row(
  children: [
    Expanded(child: Widget1()),   // MUST fill remaining space
    Flexible(child: Widget2()),   // CAN be smaller than available
    Widget3(),                    // Fixed/intrinsic size
  ],
)

// Proportional distribution
Row(
  children: [
    Flexible(flex: 2, child: Widget1()), // 2/3 of space
    Flexible(flex: 1, child: Widget2()), // 1/3 of space
  ],
)
```

### Layout Planning

Flutter layout follows three rules:
1. Constraints go **down** (parent tells child max/min size)
2. Sizes go **up** (child reports its chosen size)
3. Parent sets **position**

Key patterns:

```dart
// Full-width with padding
Container(
  width: double.infinity,
  padding: const EdgeInsets.all(16),
  child: child,
)

// Stack with positioned overlays
Stack(
  children: [
    BackgroundWidget(),
    Positioned(top: 20, left: 20, child: Overlay()),
    Positioned.fill(child: FullOverlay()),
    Align(alignment: Alignment.bottomRight, child: Corner()),
  ],
)
```

### Design System Mapping

#### Material 3 Theme Setup

```dart
MaterialApp(
  theme: ThemeData(
    useMaterial3: true,
    colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF6750A4)),
  ),
)

// Accessing theme values in widgets
Theme.of(context).colorScheme.primary
Theme.of(context).colorScheme.primaryContainer
Theme.of(context).textTheme.headlineMedium
```

#### Custom Design Tokens

```dart
class AppColors {
  static const primary = Color(0xFF6200EE);
  static const surface = Color(0xFFF5F5F5);
}

class AppSpacing {
  static const xs = 4.0;
  static const sm = 8.0;
  static const md = 16.0;
  static const lg = 24.0;
  static const xl = 32.0;
}

// Usage
Padding(
  padding: const EdgeInsets.all(AppSpacing.md),
  child: Text('Hello', style: TextStyle(color: AppColors.primary)),
)
```

### Responsive Design Patterns

```dart
// MediaQuery breakpoints
final width = MediaQuery.of(context).size.width;
final isPhone = width < 600;
final isTablet = width >= 600 && width < 1200;
final isDesktop = width >= 1200;

// LayoutBuilder for constraint-based responsiveness
LayoutBuilder(
  builder: (context, constraints) {
    final columns = (constraints.maxWidth / 300).floor().clamp(1, 4);
    return GridView.builder(
      gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
        crossAxisCount: columns,
      ),
      itemBuilder: (context, index) => GridItem(index: index),
    );
  },
)
```

---

## 2. Code Generation

### Performance-First Approach

Always use `const` constructors for static content. This prevents unnecessary
widget rebuilds during the framework's reconciliation pass.

```dart
// Use const for all static widgets
const Text('Static text')
const SizedBox(height: 16)
const EdgeInsets.all(16)
const Icon(Icons.home)

// Mark custom widgets as const-constructable
class ProductList extends StatelessWidget {
  const ProductList({super.key});

  @override
  Widget build(BuildContext context) {
    return const Column(
      children: [
        _Header(),
        _Content(),
      ],
    );
  }
}

// Use ValueKey for stateful widgets in lists
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) {
    return ProductCard(
      key: ValueKey(items[index].id),
      product: items[index],
    );
  },
)
```

### Styling with Material 3 and BoxDecoration

#### BoxDecoration (the main styling workhorse)

```dart
Container(
  padding: const EdgeInsets.all(20),
  decoration: BoxDecoration(
    color: Colors.white,
    borderRadius: BorderRadius.circular(16),
    boxShadow: [
      BoxShadow(
        color: Colors.black.withOpacity(0.1),
        blurRadius: 10,
        offset: const Offset(0, 4),
      ),
    ],
    gradient: const LinearGradient(
      begin: Alignment.topLeft,
      end: Alignment.bottomRight,
      colors: [Color(0xFF6750A4), Color(0xFF9C4DCC)],
    ),
    border: Border.all(color: Colors.grey[300]!, width: 1),
  ),
  child: child,
)
```

#### Rounded Images

```dart
ClipRRect(
  borderRadius: BorderRadius.circular(12),
  child: Image.network(url, width: 120, height: 120, fit: BoxFit.cover),
)
```

#### Typography

```dart
Text(
  'Title',
  style: const TextStyle(
    fontSize: 24,
    fontWeight: FontWeight.w700,   // w400=regular, w500=medium, w600=semi, w700=bold
    height: 1.3,                   // line-height multiplier
    letterSpacing: -0.5,
    color: Color(0xFF1A1A1A),
  ),
  maxLines: 2,
  overflow: TextOverflow.ellipsis,
)
```

### Responsive Layout Implementation

```dart
class ResponsiveLayout extends StatelessWidget {
  const ResponsiveLayout({super.key});

  @override
  Widget build(BuildContext context) {
    final size = MediaQuery.of(context).size;
    final padding = MediaQuery.of(context).padding;

    if (size.width < 600) return const PhoneLayout();
    if (size.width < 1200) return const TabletLayout();
    return const DesktopLayout();
  }
}

// Adaptive grid that recalculates columns from constraints
class AdaptiveGrid extends StatelessWidget {
  const AdaptiveGrid({super.key});

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        final columns = (constraints.maxWidth / 300).floor().clamp(1, 4);
        return GridView.builder(
          gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
            crossAxisCount: columns,
            crossAxisSpacing: 16,
            mainAxisSpacing: 16,
            childAspectRatio: 0.75,
          ),
          itemBuilder: (context, index) => GridItem(index: index),
        );
      },
    );
  }
}
```

### Animations

#### Implicit Animations (simplest, state-driven)

```dart
// AnimatedContainer -- animates any property change
AnimatedContainer(
  duration: const Duration(milliseconds: 300),
  curve: Curves.easeInOut,
  width: isExpanded ? 200 : 100,
  height: isExpanded ? 200 : 100,
  color: isExpanded ? Colors.blue : Colors.red,
  child: child,
)

// AnimatedOpacity -- fade in/out
AnimatedOpacity(
  opacity: isVisible ? 1.0 : 0.0,
  duration: const Duration(milliseconds: 500),
  child: child,
)

// AnimatedCrossFade -- swap between two widgets
AnimatedCrossFade(
  firstChild: const Icon(Icons.play_arrow),
  secondChild: const Icon(Icons.pause),
  crossFadeState: isPlaying
      ? CrossFadeState.showSecond
      : CrossFadeState.showFirst,
  duration: const Duration(milliseconds: 200),
)
```

#### Explicit Animations (full control with AnimationController)

```dart
class _AnimatedWidgetState extends State<AnimatedWidget>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _animation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      duration: const Duration(seconds: 2),
      vsync: this,
    );
    _animation = Tween<double>(begin: 0, end: 1).animate(
      CurvedAnimation(parent: _controller, curve: Curves.easeInOut),
    );
    _controller.repeat(reverse: true);
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _animation,
      builder: (context, child) {
        return Opacity(opacity: _animation.value, child: child);
      },
      child: const Icon(Icons.favorite, size: 100),
    );
  }
}
```

#### Hero Animations (shared-element page transitions)

```dart
// Source page
Hero(
  tag: 'product-image-${product.id}',
  child: Image.network(product.imageUrl),
)

// Destination page -- same tag triggers the animation
Hero(
  tag: 'product-image-${product.id}',
  child: Image.network(product.imageUrl),
)
```

### Accessibility

```dart
// Semantic labels for icons and images
IconButton(
  icon: const Icon(Icons.favorite),
  tooltip: 'Add to favorites',    // serves as semantic label
  onPressed: () {},
)

Semantics(
  label: 'Product image',
  child: Image.network(imageUrl),
)

// Exclude purely decorative elements
ExcludeSemantics(
  child: Container(decoration: BoxDecoration(/* decorative only */)),
)

// Merge semantics so screen readers announce as one unit
MergeSemantics(
  child: Row(children: [Icon(Icons.star), Text('4.5'), Text('rating')]),
)

// Custom semantic properties
Semantics(
  label: 'Product rating 4.5 out of 5',
  hint: 'Tap to see reviews',
  child: ratingWidget,
)

// Minimum touch target: 48x48 logical pixels
InkWell(
  onTap: () {},
  child: Container(
    width: 48,
    height: 48,
    alignment: Alignment.center,
    child: const Icon(Icons.add),
  ),
)
```

---

## 3. Visual Comparison Workflow

This section documents the ImageMagick-based workflow for comparing a Flutter
implementation screenshot against the original design. This is specialized
procedural knowledge not available from general Flutter documentation.

### Environment Setup

```bash
# 1. Ensure ImageMagick is available
which magick || which convert || \
  (echo "Installing ImageMagick..." && \
   apt-get update -qq && apt-get install -y -qq imagemagick > /dev/null 2>&1)

# 2. Create working directories
mkdir -p /tmp/ui-comparison/{regions-design,regions-impl,diffs}
```

### The Crop-Zoom-Compare Methodology

This is the core technique. Never rely on full-size images alone -- Claude's
vision is far more accurate on cropped, enlarged regions.

#### Step 1: Validate and Normalize Dimensions

```bash
# Get dimensions of both images
identify -format "%wx%h" design.png
identify -format "%wx%h" screenshot.png

# If dimensions differ, resize implementation to match design
convert screenshot.png -resize WxH! screenshot_resized.png
```

#### Step 2: View Full Images, Identify Regions

Use the Read tool to view both full images. List every visible component region
with estimated bounding boxes (x, y, width, height).

Standard regions to always check on a mobile screen:
1. Status bar / App bar (top)
2. Navigation (tabs, bottom nav)
3. Hero / Header area
4. Content cards / List items (repeating)
5. Text blocks (titles, subtitles, body)
6. Buttons / CTAs
7. Input fields
8. Icons (size, color, alignment)
9. Images (size, aspect ratio, border radius)
10. Footer / Bottom area
11. Spacing between sections (vertical rhythm)

#### Step 3: Crop Each Region from Both Images

```bash
# Syntax: convert input.png -crop WxH+X+Y +repage output.png
# The +repage resets the virtual canvas after cropping

# Example: header region (top 200px of a 400px-wide image)
convert design.png -crop 400x200+0+0 +repage /tmp/ui-comparison/regions-design/header.png
convert screenshot.png -crop 400x200+0+0 +repage /tmp/ui-comparison/regions-impl/header.png

# Example: a button at position (100, 500) with size 200x60
convert design.png -crop 200x60+100+500 +repage /tmp/ui-comparison/regions-design/button.png
convert screenshot.png -crop 200x60+100+500 +repage /tmp/ui-comparison/regions-impl/button.png
```

#### Step 4: Zoom In (2-4x Enlargement)

```bash
# Scale up 3x for pixel-level inspection
convert /tmp/ui-comparison/regions-design/header.png \
  -scale 300% /tmp/ui-comparison/regions-design/header_zoomed.png
convert /tmp/ui-comparison/regions-impl/header.png \
  -scale 300% /tmp/ui-comparison/regions-impl/header_zoomed.png
```

Then use the Read tool to view each zoomed pair and compare carefully.

#### Step 5: Generate Pixel Diff for Each Region

```bash
# Highlight differences in red; output pixel count of differing pixels
compare -metric AE -fuzz 5% \
  /tmp/ui-comparison/regions-design/header.png \
  /tmp/ui-comparison/regions-impl/header.png \
  /tmp/ui-comparison/diffs/header_diff.png 2>&1
```

The `-fuzz 5%` tolerance prevents reporting sub-perceptual color differences.
The AE metric returns the absolute count of differing pixels.

#### Step 6: Run Overall SSIM (Optional Summary Metric)

```bash
compare -metric SSIM design.png screenshot.png null: 2>&1
```

SSIM (Structural Similarity Index) returns a value from 0 to 1, where 1 means
the images are identical. Values above 0.95 generally indicate high fidelity.

### Color Sampling with ImageMagick

```bash
# Get the exact hex color at a specific pixel coordinate (x=150, y=300)
convert design.png -crop 1x1+150+300 -format "%[hex:u.p{0,0}]" info:
convert screenshot.png -crop 1x1+150+300 -format "%[hex:u.p{0,0}]" info:

# Get the average color of a 50x50 region starting at (100, 200)
convert design.png -crop 50x50+100+200 -scale 1x1! -format "%[hex:u.p{0,0}]" info:
convert screenshot.png -crop 50x50+100+200 -scale 1x1! -format "%[hex:u.p{0,0}]" info:
```

This lets you make precise statements like "Background is #F8F8F8, should be
#FFFFFF" instead of vague "color looks off."

### Component Analysis Checklist

When examining each cropped region, check every property:

| Property       | What to Check                                              |
|----------------|-------------------------------------------------------------|
| **Colors**     | Background, text, borders, icons, shadows                   |
| **Typography** | Font size, weight, family, line height, letter spacing      |
| **Spacing**    | Padding (inner), margins (outer), gaps between children     |
| **Borders**    | Width, color, radius, style (solid/dashed)                  |
| **Shadows**    | Offset X/Y, blur radius, spread, color/opacity             |
| **Size**       | Component width, height, aspect ratio                       |
| **Alignment**  | Horizontal/vertical position, text alignment                |
| **Content**    | Correct icons, images, text content, asset sizes            |

### Comparison Report Format

The final output follows this structure:

```
OVERALL METRICS
  SSIM Score:    0.XXX
  Pixel Diff:    X.X%

COMPONENT-BY-COMPONENT ANALYSIS
  [COMPONENT NAME] (region: X,Y -> WxH)
    Observations:
      - Background is #F8F8F8, should be #FFFFFF
      - Padding is 12px, should be 16px
    Fix:
      // Current code -> Fixed code (dart snippet)

PRIORITIZED FIX LIST
  HIGH PRIORITY   (visually obvious to users)
  MEDIUM PRIORITY (noticeable on inspection)
  LOW PRIORITY    (minor polish)

FLUTTER CODE FIXES
  Complete, copy-pasteable code snippets for every fix
```

Key principles for the report:
- **Always zoom.** Never rely only on full-size images.
- **Be specific.** Use exact hex values and pixel measurements.
- **Provide code.** Every issue needs a concrete Flutter code fix.
- **Prioritize by visual impact.** What a user notices first is HIGH priority.
- **Measure, don't guess.** Use ImageMagick to extract actual values.
- **Check both directions.** Implementation may have extras the design lacks.

---

## 4. Iteration Workflow

### Four-Phase Pipeline

The design-to-implementation process runs through four sequential phases,
repeating phases 2-4 until fidelity exceeds 95%.

| Phase | Purpose               | Input                        | Output                          |
|-------|-----------------------|------------------------------|---------------------------------|
| 1     | Design Analysis       | Design file (Figma/screenshot)| Widget hierarchy, design tokens |
| 2     | Code Generation       | Widget hierarchy + any fixes | Flutter code                    |
| 3     | Screenshot Capture    | Running app on device/sim    | Screenshot PNG                  |
| 4     | Visual Comparison     | Design + screenshot          | Fidelity score, fix list        |

**Flow:** Phase 1 runs once. Phases 2-3-4 repeat until fidelity >= 95% or max
iterations reached.

Each phase depends on the previous one's output. They must run sequentially.
The only parallelism opportunity is when processing multiple independent screens
(run Phase 1 for all screens in parallel, then process each through 2-3-4).

### Iteration Management Rules

| Scenario                          | Action                                    |
|-----------------------------------|-------------------------------------------|
| Fidelity >= 95%                   | Complete. Generate final report.           |
| Fidelity < 95%, iterations < 10  | Apply fixes from comparison, iterate.      |
| No improvement for 2 iterations   | Escalate to user with details.             |
| Iterations >= 10                  | Escalate with current fidelity and options.|

When escalating, present:
- Current fidelity score
- Remaining unresolved issues
- What has been tried
- Options for the user (accept current state, manually adjust, provide
  clarification on ambiguous design elements)

### Fix Prioritization Scoring

When multiple issues are identified, prioritize fixes by their impact on the
overall fidelity score:

| Category                                          | Fidelity Impact |
|---------------------------------------------------|-----------------|
| Structural (wrong widgets, missing components)    | 10-15 points    |
| Spacing and colors                                | 5-10 points     |
| Typography (size, weight, family)                 | 3-5 points      |
| Polish (borders, shadows, subtle decoration)      | 1-3 points      |

Address highest-impact fixes first in each iteration to converge fastest.

### Error Handling Strategies

| Error Type            | Strategy                                               |
|-----------------------|--------------------------------------------------------|
| Agent/phase failure   | Check input validity, retry with corrections, escalate |
| Stuck iterations      | Review last 3 reports for recurring issues, escalate   |
| Design ambiguity      | Document assumptions, flag for user review              |
| Max iterations hit    | Present current state and options to user               |

### Multi-Screen Designs

For designs with multiple screens:

1. Run Phase 1 (Design Analysis) for all screens in parallel.
2. Identify shared components (headers, buttons, cards) across screens.
3. Implement shared components first to ensure cross-screen consistency.
4. Process each screen through Phases 2-3-4 individually.

### Success Criteria

An implementation is considered complete when:
1. Fidelity score >= 95%
2. All HIGH priority issues from the comparison report are resolved
3. Code uses const constructors, proper keys, and follows performance best practices
4. Screenshots are captured and archived for documentation
5. A final report is generated listing deliverables

---

## Quick Reference: Common Flutter Fixes

These are the fixes most frequently needed during design-implementation
iterations.

### Colors

```dart
// Color format: 0xAARRGGBB (AA = alpha/opacity)
Container(color: const Color(0xFFFFFFFF))  // white, fully opaque
Container(color: const Color(0x80000000))  // black, 50% opacity
```

### Spacing

```dart
Padding(padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12))
Container(margin: const EdgeInsets.only(bottom: 8))
const SizedBox(height: 8)   // vertical gap in Column
const SizedBox(width: 12)   // horizontal gap in Row
```

### Border Radius

```dart
// On Container via decoration
Container(
  decoration: BoxDecoration(
    borderRadius: BorderRadius.circular(12),
  ),
)

// Asymmetric
BorderRadius.only(topLeft: Radius.circular(12), topRight: Radius.circular(12))

// On images
ClipRRect(
  borderRadius: BorderRadius.circular(12),
  child: Image.network(url, fit: BoxFit.cover),
)
```

### Shadows

```dart
BoxShadow(
  color: Colors.black.withOpacity(0.1),
  offset: const Offset(0, 4),
  blurRadius: 12,
  spreadRadius: 0,
)
```

### Font Weight Reference

| Constant             | CSS Equivalent |
|----------------------|----------------|
| `FontWeight.w400`   | regular        |
| `FontWeight.w500`   | medium         |
| `FontWeight.w600`   | semibold       |
| `FontWeight.w700`   | bold           |
