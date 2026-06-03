---
title: Building AI-Style Loading Animations in SwiftUI
description: Shimmer text and a worm-style dot grid from the AI Animation Demo iOS app, with SwiftUI code walkthroughs.
pubDatetime: 2026-06-03T15:10:00Z
tags:
  - swiftui
  - ios
  - animation
  - shimmer
---

Modern AI apps communicate state with subtle motion: a shimmer across “Thinking…”, a pulsing orb while analyzing, or a dot grid that suggests progress without a traditional spinner. This post breaks down two patterns from the open-source **[AI Animation Demo](https://github.com/mdo91/AI-Animation-Demo)** iOS app — **shimmering text** and a **worm-style dot grid** — and walks through the SwiftUI code that powers them.

**Repository:** https://github.com/mdo91/AI-Animation-Demo

---

## Project overview

The app is a SwiftUI playground: a navigation list of animation demos, each backed by a reusable component. Relevant structure:

```
Components/
├── Shimmer/
│   ├── ShimmeringText.swift      # Core shimmer engine
│   ├── GrokShimmerText.swift     # Vibrant gradient preset
│   └── ChatGPTShimmerText.swift  # Monochrome thinking preset
└── DotGrid/
    └── SequentialDotGrid.swift   # Worm dot loader
```

Both animations use **`TimelineView(.animation)`** instead of `@State` + `withAnimation` loops. That gives frame-accurate timing, avoids visible “snap” at loop boundaries, and keeps motion smooth when the system is under load.

---

## Part 1: Shimmering text animation

### What it looks like

A gradient band sweeps across text. Depending on configuration, the effect can:

- **Fill the glyphs** with a moving gradient (Grok / ChatGPT style)
- **Highlight secondary gray text** with a soft white band (summarizing / processing labels)

### The `ShimmeringText` view structure

The full component wires timing, layout, and rendering together:

```swift
struct ShimmeringText: View {
    let text: String
    var font: Font = .system(.title2, design: .rounded).weight(.bold)
    var textColor: Color = .clear
    var gradientColors: [Color] = [.blue, .purple, .pink, .blue]
    var duration: TimeInterval = 2.0
    var repeatDelay: TimeInterval = 0

    @Environment(\.colorScheme) private var colorScheme

    private var usesBaseTextColor: Bool {
        textColor != .clear
    }

    var body: some View {
        TimelineView(.animation) { context in
            let phase = phase(at: context.date)

            Text(text)
                .font(font)
                .foregroundColor(textColor)
                .fixedSize(horizontal: false, vertical: true)
                .overlay(
                    GeometryReader { geometry in
                        shimmerGradient(in: geometry, phase: phase)
                    }
                    .mask(
                        Text(text)
                            .font(font)
                            .fixedSize(horizontal: false, vertical: true)
                    )
                    .blendMode(usesBaseTextColor ? .plusLighter : .normal)
                )
        }
    }
    // shimmerGradient, phase(at:), overlayHighlightColors …
}
```

**Why each piece matters:**

| Piece | Role |
|---|---|
| `TimelineView(.animation)` | Recomputes `phase` every frame from wall-clock time — no `@State` timer loops |
| `textColor` + `usesBaseTextColor` | Switches between full gradient fill (`.clear`) and overlay highlight (e.g. `.secondary`) |
| `GeometryReader` | Supplies the **measured text width** so offset math matches real glyph bounds |
| Duplicate `Text` in `.mask()` | Clips the gradient to letter shapes only (SwiftUI’s text-mask pattern) |
| `.fixedSize(horizontal: false, vertical: true)` | Keeps mask and base text the same size when labels wrap to multiple lines |
| `.blendMode(.plusLighter)` | In overlay mode, **adds** light on top of gray text instead of painting opaque color |

SwiftUI has no `background-clip: text` like CSS. The mask is what makes color appear *inside* the glyphs.

### Driving animation with `phase(at:)`

```swift
/// Returns 0→1 during the sweep, then holds at 0 during `repeatDelay` (rest position).
private func phase(at date: Date) -> CGFloat {
    let cycleLength = duration + repeatDelay
    guard cycleLength > 0 else { return 0 }

    let cyclePosition = date.timeIntervalSinceReferenceDate
        .truncatingRemainder(dividingBy: cycleLength)

    guard cyclePosition < duration else { return 0 }

    return CGFloat(cyclePosition / duration)
}
```

Think of one cycle as two segments:

```
|←—— duration (phase 0→1) ——→|← repeatDelay (phase = 0) →|
```

- During **`duration`**, `phase` linearly goes from `0` to `1` — that drives the gradient sweep.
- During **`repeatDelay`**, `phase` stays at `0` — the highlight rests **off-screen left**, so the next sweep starts from the same visual state (no snap).

`truncatingRemainder(dividingBy:)` loops the timeline forever without resetting views.

### Deep dive: `shimmerGradient`

This is the heart of the effect. It receives a `GeometryProxy` (text bounds) and the current `phase`, then builds **one of two** moving gradients:

```swift
@ViewBuilder
private func shimmerGradient(in geometry: GeometryProxy, phase: CGFloat) -> some View {
    let width = geometry.size.width

    if usesBaseTextColor {
        let bandWidth = max(width * 0.6, 24)

        LinearGradient(
            colors: overlayHighlightColors,
            startPoint: .leading,
            endPoint: .trailing
        )
        .frame(width: bandWidth, height: geometry.size.height, alignment: .leading)
        .offset(x: -bandWidth + phase * (width + bandWidth))
    } else {
        LinearGradient(
            colors: gradientColors,
            startPoint: .leading,
            endPoint: .trailing
        )
        .frame(width: width * 3, height: geometry.size.height, alignment: .leading)
        .offset(x: -width * 2 + phase * width * 2)
    }
}
```

`@ViewBuilder` lets the function return different gradient configurations from one place. The branch is controlled by **`usesBaseTextColor`** (`textColor != .clear`).

---

#### Branch A — Overlay highlight mode (`usesBaseTextColor == true`)

Used for calm status labels: gray `.secondary` text with a soft white band passing over it (summarizing / processing).

**Gradient shape** — a narrow band, not a full rainbow:

```swift
private var overlayHighlightColors: [Color] {
    let peak = colorScheme == .dark
        ? Color.white.opacity(0.35)
        : Color.white.opacity(0.35)
    return [.clear, peak, .clear]
}
```

`[.clear, peak, .clear]` creates a smooth spotlight: transparent edges, bright center. The peak opacity is tuned for both light and dark mode (here `0.35` on white).

**Band width** — `max(width * 0.6, 24)`:

- On long strings, the highlight is ~60% of text width (visible sweep).
- On short labels (“Thinking…”), minimum `24pt` so the band never looks like a tiny dot.

**Offset math** — `offset(x: -bandWidth + phase * (width + bandWidth))`:

| `phase` | `x` offset | Visual |
|---|---|---|
| `0` | `-bandWidth` | Band fully **left** of text (hidden) |
| `0.5` | `width/2 - bandWidth/2` | Band crosses **center** |
| `1` | `width` | Band fully **right** of text (hidden) |

At both endpoints the band is off the glyphs → **seamless loop** when `repeatDelay` holds `phase` at `0`.

Combined with `.blendMode(.plusLighter)`, the band **brightens** existing gray pixels rather than replacing them — closer to native iOS skeleton/shimmer on secondary labels.

---

#### Branch B — Full-fill mode (`textColor == .clear`)

Used for Grok-style (colorful) and ChatGPT-style (monochrome) “thinking” labels where the **gradient is the text**.

**Wider gradient** — `frame(width: width * 3)`:

The gradient is three times the text width so a multi-stop palette (blue → purple → pink → blue) always covers every glyph during the sweep, even at the edges.

**Offset math** — `offset(x: -width * 2 + phase * width * 2)`:

| `phase` | `x` offset | Visual |
|---|---|---|
| `0` | `-2×width` | Gradient parked left; only the **right third** may clip into the mask |
| `0.5` | `-width` | Gradient centered on text |
| `1` | `0` | Gradient shifted right; sweep completes |

The total travel distance is `2×width`, matching a full left-to-right pass across the text box. Because the gradient strip is `3×width` wide, there is no empty gap inside the mask mid-animation.

**Blend mode** — `.normal` (default): the masked gradient *is* the visible text color; no base layer underneath.

---

#### Why `GeometryReader` wraps only the gradient

```swift
.overlay(
    GeometryReader { geometry in
        shimmerGradient(in: geometry, phase: phase)
    }
    .mask(Text(text) …)
)
```

`Text` sizes itself to its content. The overlay does not automatically inherit that size for layout purposes — `GeometryReader` expands to fill the text’s laid-out frame and exposes `geometry.size.width`. Without it, you would be guessing widths and the shimmer would desync on different strings or Dynamic Type sizes.

---

#### End-to-end data flow (one frame)

```
context.date
    → phase(at:) → CGFloat 0…1
    → shimmerGradient(geometry, phase)
        → LinearGradient + frame + offset
    → .mask(Text)  → clip to glyphs
    → .blendMode   → plusLighter or normal
    → composited on base Text
```

Every frame, `TimelineView` repeats this pipeline. Changing `duration`, `repeatDelay`, `gradientColors`, or `textColor` swaps behavior without touching the animation engine.

### Preset wrappers

**Grok-style** — bold, colorful, continuous:

```swift
struct GrokShimmerText: View {
    let text: String

    var body: some View {
        ShimmeringText(
            text: text,
            font: ShimmeringText.prominentLabelFont,
            textColor: .clear,
            gradientColors: [.blue, .purple, .pink, .blue],
            duration: 2.0,
            repeatDelay: 0
        )
    }
}
```

**ChatGPT-style** — understated gray sweep on medium subheadline:

```swift
struct ChatGPTShimmerText: View {
    let text: String

    var body: some View {
        ShimmeringText(
            text: text,
            font: ShimmeringText.thinkingLabelFont,
            textColor: .clear,
            gradientColors: [
                ShimmeringText.statusLabelColor,
                ShimmeringText.statusLabelColor.opacity(0.45),
                ShimmeringText.statusLabelColor,
                ShimmeringText.statusLabelColor.opacity(0.45),
                ShimmeringText.statusLabelColor
            ],
            duration: 2.0,
            repeatDelay: 0
        )
    }
}
```

**Summarizing label** — calm, with pause between sweeps:

```swift
ShimmeringText(
    text: "Summerizing the text…",
    font: ShimmeringText.calmStatusFont,
    textColor: ShimmeringText.statusLabelColor,
    duration: ShimmeringText.neutralStatusDuration,      // 2.0s
    repeatDelay: ShimmeringText.neutralStatusRepeatDelay // 0.5s pause
)
```

Typography follows iOS HIG via the `ShimmeringText` extension — shared fonts and timing constants so presets stay consistent:

```swift
extension ShimmeringText {
    static let statusLabelColor = Color.secondary
    static let calmStatusFont: Font = .subheadline.weight(.regular)
    static let thinkingLabelFont: Font = .subheadline.weight(.medium)
    static let prominentLabelFont: Font = .system(.headline, design: .rounded).weight(.semibold)
    static let neutralStatusDuration: TimeInterval = 2.0
    static let neutralStatusRepeatDelay: TimeInterval = 0.5
    // …
}
```

### Shimmer tuning cheat sheet

| Parameter | Effect |
|---|---|
| `duration` | Sweep speed (e.g. 2.0s calm, 1.5s active) |
| `repeatDelay` | Pause between sweeps; `0` = continuous |
| `gradientColors` | Full-fill palette (Grok, ChatGPT) |
| `textColor` | `.clear` = gradient fill; `.secondary` = overlay highlight |
| `font` | Visual hierarchy for the label |

---

## Part 2: Worm dot grid animation

### What it looks like

Nine gray dots in a 3×3 grid. A **black “head”** moves through a custom path; trailing dots fade out like a **worm tail**, then the cycle pauses and restarts.

Grid numbering (animation order 1 → 9):

```
3  2  1
9  8  7
4  5  6
```

So the worm travels: top-right → top-center → top-left → bottom-left → … → middle-left.

### Why not animate each dot independently?

An early version animated one dot at a time: gray → black → fade → gray. That felt choppy, and loop resets were visible.

The worm pattern is better because:

- One **moving head position** drives all dots
- Trailing dots share a **fade curve** based on distance from the head
- Opacity is spatial (distance-based), not per-dot keyframes

This matches common loader patterns where staggered opacity creates a traveling wave.

### Grid layout in code

Serial numbers are stored in row/column order:

```swift
private static let gridSerialNumbers: [[Int]] = [
    [3, 2, 1],
    [9, 8, 7],
    [4, 5, 6]
]
```

Each cell looks up its serial number and asks: *how far am I behind the current head?*

### Timeline-driven head position

```swift
TimelineView(.animation) { context in
    let cyclePosition = context.date.timeIntervalSinceReferenceDate
        .truncatingRemainder(dividingBy: cycleDuration)
    let headPosition = headPosition(for: cyclePosition)

    VStack(spacing: spacing) {
        ForEach(0..<3, id: \.self) { row in
            HStack(spacing: spacing) {
                ForEach(0..<3, id: \.self) { column in
                    let serial = Self.gridSerialNumbers[row][column]
                    dotView(serial: serial, headPosition: headPosition)
                }
            }
        }
    }
}
```

Head position is a continuous float from `0` (at dot 1) to `9` (past dot 9):

```swift
private func headPosition(for cyclePosition: TimeInterval) -> Double? {
    let activeWindow = 9 * dotDuration
    guard cyclePosition < activeWindow else { return nil }
    return cyclePosition / dotDuration
}
```

After the worm completes, `headPosition` is `nil` for the pause period (`pauseBetweenCycles`).

### Rendering each dot: base + overlay

Each dot is a `ZStack` — gray base circle, black overlay with computed opacity:

```swift
private func dotView(serial: Int, headPosition: Double?) -> some View {
    let distance = trailDistance(for: serial, headPosition: headPosition)
    let opacity = overlayOpacity(for: distance)
    let scale = activeScale(for: distance)

    return ZStack {
        Circle()
            .fill(Self.originalGray)

        Circle()
            .fill(.black)
            .opacity(opacity)
            .scaleEffect(scale)
    }
    .frame(width: dotSize, height: dotSize)
}
```

**Important:** opacity is applied to a **black overlay on top of gray**, not by animating the fill color directly. That gives a clean tail fade and avoids muddy intermediate grays.

### Trail distance from the head

```swift
private func trailDistance(for serial: Int, headPosition: Double?) -> Double? {
    guard let headPosition else { return nil }

    let index = Double(serial - 1)
    let distance = headPosition - index
    guard distance >= 0, distance <= trailLength else { return nil }
    return distance
}
```

- `distance == 0` → this dot is the head (full opacity)
- `distance == 1.5` → midway in the tail (partial opacity)
- `distance > trailLength` → not in the worm yet / already faded out

### Opacity and scale curves

```swift
private func overlayOpacity(for distance: Double?) -> Double {
    guard let distance else { return 0 }

    let normalized = max(0, 1 - distance / trailLength)
    return pow(normalized, 1.2)
}

private func activeScale(for distance: Double?) -> CGFloat {
    guard let distance else { return 1.0 }
    let normalized = max(0, 1 - distance / trailLength)
    return 0.94 + CGFloat(normalized) * 0.16
}
```

- **`trailLength`** (default `2.8`) controls worm thickness — how many dots stay lit behind the head
- **`pow(..., 1.2)`** makes the tail fall off slightly faster than linear
- **Scale** gives the head a subtle pop without distracting motion

### Worm tuning cheat sheet

| Parameter | Default | Effect |
|---|---|---|
| `dotSize` | `5` | Dot diameter |
| `spacing` | `5` | Gap between dots |
| `dotDuration` | `0.16` | Time per dot step |
| `trailLength` | `2.8` | Tail length (in dot units) |
| `pauseBetweenCycles` | `0.4` | Rest before restart |

---

## Putting it together in the app

Both components plug into the same demo shell:

```swift
struct SequentialDotGridDemoView: View {
    var body: some View {
        AnimationDemoLayout(
            title: "Sequential dot grid",
            description: "Nine gray dots animate as a worm pattern in order 1→9."
        ) {
            SequentialDotGrid()
                .padding(24)
        }
    }
}
```

The animation list is driven by `AnimationKind`, an enum that maps each demo title to its view — easy to extend with new loaders.

---

## Key takeaways

1. **Mask + gradient overlay** is the SwiftUI equivalent of text shimmer on the web.
2. **`TimelineView`** beats manual animation loops for continuous, seamless motion.
3. **Two shimmer modes** — full gradient fill vs. overlay highlight — cover most AI status label use cases.
4. **Worm loaders** work best with a single traveling head + distance-based opacity tail, not independent per-dot flashes.
5. **Typography matters** — status labels should use `.subheadline` and `Color.secondary`, not headline-sized bold text.

---

## Try it yourself

Clone the repo, run it in Xcode on an iPhone simulator, and browse the animation list:

**https://github.com/mdo91/AI-Animation-Demo**

MIT licensed — reuse the components, tweak timing and colors, or add your own patterns. Contributions welcome.
