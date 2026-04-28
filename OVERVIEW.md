# FractalClockSaver - Codebase Overview

## Project Summary

**FractalClockSaver** is a macOS screen saver built with Swift and AppKit, based on the original [HackerPoet/FractalClock](https://github.com/HackerPoet/FractalClock) concept. It renders an animated clock where the clock hands recursively branch into smaller fractal copies of themselves, creating a visually striking fractal tree effect. The project targets macOS 10.14 Mojave and above.

---

## Architecture

The project follows the standard macOS ScreenSaver framework architecture with a clean separation between the rendering view and the configuration UI.

```
+-----------------------------------------------------+
|                  macOS ScreenSaver                   |
|                    Framework                         |
+-----------------------------------------------------+
|                                                     |
|  +----------------------+    +-------------------+  |
|  |   FractalClockView   |    |  ConfigureSheet  |  |
|  |   (ScreenSaverView)  |<---|    Controller    |  |
|  |                      |    |   (NSObject)     |  |
|  |  - animateOneFrame() |    |                  |  |
|  |  - draw()            |    |  - Preferences   |  |
|  |  - drawHands()       |    |    .xib (UI)     |  |
|  |  - recursiveDraw()   |    |                  |  |
|  |  - drawTicks()       |    |  - UserDefaults  |  |
|  |  - drawNumbers()     |    |    persistence   |  |
|  |  - drawFlavourText() |    |                  |  |
|  +----------------------+    +-------------------+  |
|                                                     |
+-----------------------------------------------------+
```

### Directory Structure

```
FractalClockSaver/
+-- FractalClock.xcodeproj/          # Xcode project configuration
+-- FractalClock/
|   +-- FractalClockView.swift       # Main screen saver view (entry point)
|   +-- ConfigureSheetController.swift # Preferences UI controller
|   +-- Preferences.xib              # Interface Builder file for config sheet
|   +-- Info.plist                   # Bundle configuration
|   +-- FractalClock-Bridging-Header.h # Objective-C bridging header (empty)
+-- README.md                        # User-facing documentation
+-- LICENSE                          # MIT License
+-- .gitignore
```

---

## Key Components

### 1. FractalClockView (`FractalClockView.swift`)

The core screen saver view, inheriting from `ScreenSaverView`. This is the principal class registered via `Info.plist` (`NSPrincipalClass`).

#### Lifecycle

| Method | Purpose |
|--------|---------|
| `init(frame:isPreview:)` | Initializes configuration from `UserDefaults`, sets up color scheme and path arrays, calculates the clock frame |
| `draw(_ rect:)` | Renders a single frame: background, clock hands, tick marks, numbers, and flavour text |
| `animateOneFrame()` | Called each frame by the ScreenSaver framework; updates defaults, angles, colors, and triggers redraw |

#### Core Properties

| Property | Type | Description |
|----------|------|-------------|
| `maximumDepth` | `Int` | Recursion depth for the fractal (default: 8) |
| `fractalType` | `Int` | Which hands participate in fractal branching (0=all, 1=no hour, 2=no second) |
| `showSeconds` | `Bool` | Whether the second hand is visible |
| `flavourText` | `String` | Custom text displayed inside the clock face |
| `colourScheme` | `[NSColor]` | Dynamically computed colors per recursion depth |
| `fractalPaths` | `[NSBezierPath]` | Bezier paths accumulated during recursive drawing |
| `secondHandAngle`, `minuteHandAngle`, `hourHandAngle` | `Double` | Current hand angles in radians |
| `clockFrame` | `NSRect` | Bounding rectangle for the clock within the view |

#### Drawing Pipeline

1. **Background**: Solid fill at 10% white (`NSColor(calibratedWhite: 0.1, alpha: 1.0)`)
2. **Fractal Hands** (`drawHands`): Recursively generates fractal paths from clock hand endpoints, with a 1-second render timeout to prevent hangs
3. **Tick Marks** (`drawTicks`): Major (every 5 minutes) and minor tick marks around the clock perimeter
4. **Numbers** (`drawNumbers`): Hour labels (12, 1, 2, ... 11) positioned radially
5. **Flavour Text** (`drawFlavourText`): Optional user-defined text centered in the lower clock face

#### Fractal Algorithm

The `recursiveDrawHands` method implements the core fractal logic:

- Each clock hand endpoint becomes the origin for a scaled-down copy of all three hands
- Scale factor per level: **0.7** (70% of parent size)
- Angles are accumulated: `handAngle + baseAngle`
- Recursion stops at `maximumDepth` or when the 1-second cutoff time is reached
- The `fractalType` setting controls which hands branch:
  - **Type 0** (Hour, Minute, Second): All three hands branch
  - **Type 1** (Minute, Second): Hour hand does not branch
  - **Type 2** (Hour, Minute): Second hand does not branch (locked when `showSeconds` is off)

#### Color Animation

Colors are computed using sinusoidal functions driven by `colourTime`:

```swift
r1 = sin(colourTime * 0.017) * 0.5 + 0.5
r2 = sin(colourTime * 0.011) * 0.5 + 0.5
r3 = sin(colourTime * 0.003) * 0.5 + 0.5
```

Each depth level gets a unique hue/saturation/brightness derived from these values, creating a smooth rainbow gradient that shifts over time. The deepest level has 50% alpha for a fading effect.

#### Time Calculation

Hand angles are computed from the current system time with nanosecond precision for smooth animation:

- **Second hand**: Full rotation per 60 seconds
- **Minute hand**: Full rotation per 60 minutes, incremented by second hand progress
- **Hour hand**: Full rotation per 12 hours, incremented by minute hand progress

---

### 2. ConfigureSheetController (`ConfigureSheetController.swift`)

Manages the screen saver's configuration panel, loading the UI from `Preferences.xib` and persisting settings via `ScreenSaverDefaults`.

#### UserDefaults Keys

| Enum Case | Raw Value | Type | Description |
|-----------|-----------|------|-------------|
| `fractalDepth` | `FCFractalDepth` | `Int` | Recursion depth (1-100) |
| `fractalType` | `FCFractalType` | `Int` | Fractal hand selection (0, 1, 2) |
| `secondHand` | `FCSecondsDisplay` | `Bool` | Show/hide second hand |
| `flavourText` | `FCFlavorText` | `String` | Custom display text |

#### UI Controls

| Outlet | Type | Purpose |
|--------|------|---------|
| `depth` | `NSTextField` | Numeric input for fractal depth |
| `depthStepper` | `NSStepper` | Stepper linked to depth field (max: 100) |
| `type` | `NSPopUpButton` | Dropdown for fractal type selection |
| `second` | `NSButton` | Checkbox to toggle second hand |
| `flavourText` | `NSTextField` | Multi-line text input for custom text |
| `instabilityText` | `NSTextField` | Dynamic warning text about depth stability |

#### Behavior

- **Second hand toggle**: When disabled, forces `fractalType` to 2 (Hour, Minute) and disables the type dropdown. Re-enabling restores the previously saved type.
- **Instability text**: Dynamically updates based on fractal type -- "Values above 8 tend to be unstable" for type 0, "Values above 14 tend to be unstable" for types 1 and 2.
- **Save**: Persists all values to `ScreenSaverDefaults` and calls `synchronize()`.

---

### 3. Preferences.xib

An Interface Builder file defining the configuration sheet UI. The layout includes:

- **Top row**: Fractal Depth label, numeric field, stepper, Clock Fractal dropdown
- **Second row**: "Show Second Hand" checkbox, instability warning text
- **Bottom area**: Flavour Text text field (multi-line)
- **Footer**: Cancel and OK buttons

The File's Owner is set to `ConfigureSheetController` with all outlets connected.

---

## Data Flow

```
User changes settings in Preferences.xib
         |
         v
ConfigureSheetController captures via @IBActions
         |
         v
Settings saved to ScreenSaverDefaults (plist)
         |
         v
FractalClockView.animateOneFrame() calls updateDefaults()
         |
         v
Properties updated -> updateAngles() -> updateColours()
         |
         v
setNeedsDisplay(bounds) triggers draw(_:)
         |
         v
Clock rendered with new configuration
```

---

## Technology Stack

| Technology | Usage |
|------------|-------|
| **Swift** | Primary language |
| **ScreenSaver.framework** | Base class `ScreenSaverView`, `ScreenSaverDefaults` |
| **AppKit** | UI components (`NSBezierPath`, `NSColor`, `NSFont`, `NSWindow`) |
| **Foundation** | `Calendar`, `Date`, `UserDefaults`, `AttributedString` |
| **Interface Builder** | `Preferences.xib` for config sheet layout |

---

## Design Decisions

### Render Time Limitation

The fractal recursion includes a **1-second cutoff** (`Date(timeIntervalSinceNow: 1)`) at multiple points to prevent the screen saver from hanging when users set excessively high depth values. Both the recursive generation and the path rendering loops check `Date() < cutoffTime` before proceeding.

### Fractional Time Coloring

The `colourTime` calculation differs based on whether seconds are shown:

- **With seconds**: Uses absolute time (nanoseconds + seconds + minutes*60 + hours*3600) for faster color cycling
- **Without seconds**: Uses a slower progression (seconds/1000 + minutes + hours*60) for subtle color shifts

### Code Attribution

- The fractal clock algorithm is adapted from [HackerPoet/FractalClock](https://github.com/HackerPoet/FractalClock)
- The tick mark and number drawing functions (`drawTicks`, `drawNumbers`) are adapted from [soffes/Clock.saver](https://github.com/soffes/Clock.saver)
