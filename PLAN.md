# Pure-Go Cross-Platform UI Toolkit: Implementation Plan (Revised)

## Architectural Principles

### 1. Platform Adapter Pattern

All platform-specific code implements a single interface. The rest of the codebase imports only abstractions.

```
┌─────────────────────────────────────────────────────────────┐
│                      Application Code                        │
├─────────────────────────────────────────────────────────────┤
│                      Widget Library                          │
│            (100% shared, zero platform imports)              │
├─────────────────────────────────────────────────────────────┤
│                      Layout Engine                           │
│                   (100% shared, pure math)                   │
├─────────────────────────────────────────────────────────────┤
│                   Rendering Abstraction                      │
│                     (Canvas interface)                       │
├──────────┬──────────┬──────────┬──────────┬────────────────┤
│ Software │  OpenGL  │  Metal   │  Vulkan  │   (backends)   │
├──────────┴──────────┴──────────┴──────────┴────────────────┤
│                   Platform Abstraction                       │
│                   (Platform interface)                       │
├─────────┬─────────┬─────────┬───────────┬──────────────────┤
│  X11    │ Wayland │  Win32  │  Cocoa    │  Android        │
└─────────┴─────────┴─────────┴───────────┴──────────────────┘
```

**Key insight**: Platform adapters are *thin*. They translate OS concepts to our abstractions, nothing more. All logic lives above.

### 2. Composition Over Inheritance

Widgets compose behaviors rather than inherit them:

```go
// Bad: Deep inheritance hierarchies
type Button struct { FocusableWidget }
type FocusableWidget struct { InteractiveWidget }
type InteractiveWidget struct { BaseWidget }

// Good: Compose capabilities
type Button struct {
    *WidgetCore           // Identity, bounds, lifecycle
    *FocusableBehavior    // Focus handling (optional embed)
    *HoverableBehavior    // Hover state tracking
    style    ButtonStyle
    onClick  func()
}
```

### 3. Shared Rendering Primitives

One software renderer serves as the reference implementation. GPU backends share the same primitive types and only differ in how they execute draw calls.

### 4. Event System Without Per-Widget Boilerplate

Events flow through a single dispatcher. Widgets declare interest via interfaces, not per-event handler registration.

---

## Phase 0: Core Type System (1 week)

Define types shared by **all** subsequent code.

### Deliverables

**`core/geometry.go`** - Point, Size, Rect, Insets, Transform (≈200 lines)
- All pure math, no platform dependencies
- Used by every other package

**`core/color.go`** - Color type with premultiplied alpha (≈100 lines)
- Conversion methods for platform formats (RGBA, BGRA, etc.)

**`core/events.go`** - Event type hierarchy (≈150 lines)
- Pointer, Key, Scroll, Focus, Window events
- Single `Event` interface with type switches (not separate handler registration)

**`core/interfaces.go`** - Core interfaces (≈100 lines)
```go
type Platform interface {
    Init() error
    Run() error
    Quit()
    CreateWindow(WindowConfig) (Window, error)
    Clipboard() Clipboard
    RunOnMain(func())
}

type Window interface {
    ID() uintptr
    SetTitle(string)
    Show()
    Close()
    Invalidate()
    SetRoot(Widget)
    Canvas() Canvas
}

type Canvas interface {
    Size() Size
    Clear(Color)
    DrawRect(Rect, Paint)
    DrawPath(Path, Paint)
    DrawText(string, Point, Font, Paint)
    DrawImage(Image, Rect)
    Save()
    Restore()
    Clip(Rect)
    Transform(Transform)
}

type Widget interface {
    Bounds() Rect
    SetBounds(Rect)
    Draw(Canvas)
    Event(Event) bool // Returns true if handled
    Children() []Widget
}
```

**Total Phase 0**: ~550 lines of shared types used everywhere.

---

## Phase 1: Platform Abstraction Layer (2 weeks)

### Goal
Define the **thinnest possible** platform interface and implement build-tag-based selection.

### Deliverables

**`platform/platform.go`** - Platform factory
```go
//go:build !windows && !darwin && !android

package platform

func New() (core.Platform, error) {
    return newX11() // or newWayland based on env
}
```

**`platform/platform_windows.go`**
```go
//go:build windows

package platform

func New() (core.Platform, error) {
    return newWin32()
}
```

Similar one-liners for darwin, android.

**Shared helpers** (`platform/shared/`):
- `eventloop.go` - Generic event loop with platform callbacks
- `timer.go` - Timer management (platform provides primitives, logic is shared)
- `cursor.go` - Cursor type mapping

### Architecture Decision
Platforms implement this minimal surface:
```go
type platformDriver interface {
    init() error
    pollEvents() []rawEvent      // Platform-native event
    createWindow(config) windowHandle
    destroyWindow(windowHandle)
    getWindowBuffer(windowHandle) []byte  // For software rendering
    presentBuffer(windowHandle)
    
    // Optional, for GPU
    createGLContext(windowHandle) glContext
    swapBuffers(windowHandle)
}
```

The shared `eventloop.go` calls these primitives. **Platform code never contains event dispatch logic.**

---

## Phase 2: Software Renderer (2 weeks)

### Goal
Single implementation that works on all platforms.

### Deliverables

**`render/software/canvas.go`** - Canvas implementation (~400 lines)
- State stack (transform, clip)
- Delegates to rasterizer

**`render/software/raster.go`** - Rasterization (~600 lines)
- Rect fill (fast path)
- Path fill (scanline)
- Path stroke (offsetting)
- Line drawing (Bresenham + Wu AA)
- Blending modes

**`render/software/text.go`** - Text rendering stub (~50 lines)
- Returns placeholder rectangles
- Full implementation in Phase 10

**`render/software/buffer.go`** - Pixel buffer (~150 lines)
- Generic RGBA buffer
- Platform format conversion on demand

**Key reuse**: The `Canvas` interface is identical for software and GPU. Widget code never knows which backend is active.

---

## Phase 3: First Platform - X11 (2 weeks)

### Goal
Minimal X11 implementation to prove the architecture.

### Deliverables

**`platform/x11/`** - ~800 lines total
- Connection management
- Window creation/destruction  
- Event translation (X11 event → core.Event)
- Shared memory buffer for rendering

### What's NOT here
- No widget logic
- No event dispatch logic
- No rendering logic
- No layout logic

The X11 package only translates between X11 protocol and our abstractions.

### Validation
Run demo app showing:
- Window appears
- Mouse/keyboard events logged
- Rectangle drawn via Canvas

---

## Phase 4: Event Dispatch (1 week)

### Goal
Single, shared event dispatcher used by all platforms.

### Deliverables

**`ui/dispatch.go`** (~200 lines)
```go
type Dispatcher struct {
    root    Widget
    focused Widget
    hovered Widget
    captured Widget
}

func (d *Dispatcher) Dispatch(e core.Event) {
    switch e := e.(type) {
    case *core.PointerEvent:
        target := d.hitTest(e.Position)
        d.deliverPointer(target, e)
    case *core.KeyEvent:
        d.deliverKey(d.focused, e)
    // ...
    }
}
```

**`ui/focus.go`** (~100 lines)
- Tab navigation
- Focus ring tracking

**`ui/hittest.go`** (~50 lines)
- Recursive bounds checking

### Reuse
- All platforms use the same dispatcher
- All widgets receive events the same way
- No per-widget event registration boilerplate

---

## Phase 5: Widget Core (2 weeks)

### Goal
Composable building blocks that widgets embed.

### Deliverables

**`widget/core.go`** - Base widget data (~150 lines)
```go
type Core struct {
    id       uint64
    parent   Widget
    bounds   Rect
    visible  bool
    enabled  bool
}

func (c *Core) Bounds() Rect { return c.bounds }
func (c *Core) SetBounds(r Rect) { c.bounds = r }
// ... minimal accessors
```

**`widget/behaviors.go`** - Reusable behaviors (~200 lines)
```go
type Focusable struct {
    focused  bool
    tabIndex int
}

func (f *Focusable) IsFocused() bool { return f.focused }
func (f *Focusable) Focus()          { f.focused = true }
func (f *Focusable) Blur()           { f.focused = false }

type Hoverable struct {
    hovered bool
}

type Pressable struct {
    pressed bool
}
```

**`widget/container.go`** - Child management (~100 lines)
```go
type Container struct {
    Core
    children []Widget
    layout   Layout
}
```

### Reuse Pattern
```go
// Button embeds shared behaviors
type Button struct {
    widget.Core
    widget.Focusable
    widget.Hoverable
    widget.Pressable
    
    label   string
    onClick func()
}

// TextField embeds different mix
type TextField struct {
    widget.Core
    widget.Focusable
    // No Hoverable, no Pressable - different interaction model
    
    text      string
    cursor    int
    selection Range
}
```

---

## Phase 6: Layout Engine (2 weeks)

### Goal
Pure math, zero platform dependencies.

### Deliverables

**`layout/constraint.go`** - Constraint types (~50 lines)
```go
type Constraints struct {
    Min, Max Size
}
```

**`layout/interface.go`** - Layout interface (~30 lines)
```go
type Layout interface {
    Measure(children []Measurable, constraints Constraints) Size
    Arrange(children []Widget, bounds Rect)
}
```

**`layout/box.go`** - Row/Column (~150 lines)
**`layout/grid.go`** - Grid (~150 lines)  
**`layout/stack.go`** - Z-order stacking (~80 lines)
**`layout/flex.go`** - Flexbox-style (~200 lines)

### Reuse
- All widgets use the same layout engine
- Layouts are pure functions on geometry
- Zero allocations in hot path (pre-size slices)

---

## Phase 7: Basic Widgets (2 weeks)

### Goal
Essential widget set using all prior infrastructure.

### Deliverables

Each widget: <100 lines because infrastructure is shared.

**`widgets/label.go`** - Text display
**`widgets/button.go`** - Clickable button  
**`widgets/textfield.go`** - Single-line input
**`widgets/checkbox.go`** - Toggle
**`widgets/container.go`** - Generic container
**`widgets/scrollview.go`** - Scrollable area

### Widget Template
```go
type Button struct {
    widget.Core       // From Phase 5
    widget.Focusable  // From Phase 5
    widget.Hoverable  // From Phase 5
    
    label   string
    style   ButtonStyle
    onClick func()
}

func (b *Button) Draw(c core.Canvas) {
    // ~15 lines: pick colors based on state, draw rect, draw text
}

func (b *Button) Event(e core.Event) bool {
    // ~20 lines: handle pointer/key, update state, call onClick
}

func (b *Button) Measure(c Constraints) Size {
    // ~5 lines: text size + padding
}
```

**Total per widget**: 40-80 lines because behaviors, events, layout are all shared.

---

## Phase 8: Windows Platform (2 weeks)

### Goal
Second platform, validating abstraction boundaries.

### Deliverables

**`platform/win32/`** - ~800 lines (same budget as X11)
- Window class registration
- Message pump integration
- Event translation
- DIB section for software rendering

### Validation
Same demo app runs on Windows without any changes to:
- Widget code
- Layout code
- Rendering code
- Event dispatch code

Only `platform/` changes.

---

## Phase 9: macOS Platform (3 weeks)

### Goal
Third platform, handling Objective-C bridge.

### Approach
Use `purego` or direct `libobjc` calls:
```go
var objc_msgSend func(obj, sel uintptr, args ...uintptr) uintptr

func init() {
    purego.RegisterLibFunc(&objc_msgSend, libobjc, "objc_msgSend")
}
```

### Deliverables

**`platform/darwin/`** - ~1200 lines
- NSApplication setup
- NSWindow creation
- NSView for rendering
- Event translation

### What's Shared
Everything except the platform package. macOS uses the same:
- Software renderer (or Metal backend later)
- Event dispatcher
- Widget library
- Layout engine

---

## Phase 10: Text Rendering (3 weeks)

### Goal
Real text, minimal dependencies.

### Approach
Embed a minimal font (Go Mono or similar) in the binary. Parse TrueType ourselves.

### Deliverables

**`text/font.go`** - Font interface (~50 lines)
**`text/truetype/`** - TrueType parser (~500 lines)
- Header parsing
- Glyph outline extraction
- Simple hinting

**`text/raster.go`** - Glyph rasterizer (~300 lines)
- Reuse path rasterizer from Phase 2
- Glyph caching

**`text/layout.go`** - Text layout (~200 lines)
- Line breaking
- Horizontal positioning
- Vertical metrics

**`text/shaper.go`** - Basic shaping (~100 lines)
- Latin script support
- Kerning

### Integration
`render/software/text.go` now calls real text renderer instead of placeholder.

---

## Phase 11: GPU Rendering Backend (3 weeks)

### Goal
Optional GPU acceleration sharing Canvas interface.

### Deliverables

**`render/gl/canvas.go`** - OpenGL Canvas (~400 lines)
- Same interface as software Canvas
- Batches draw calls
- Uses shared Path/Paint types

**`render/gl/shaders.go`** - Embedded shaders (~200 lines)
**`render/gl/atlas.go`** - Texture atlas for glyphs (~150 lines)

### Platform Integration
Each platform optionally provides GL context:
```go
type Window interface {
    // ... existing methods
    GLContext() GLContext  // nil if not available
}
```

Canvas selection:
```go
func (w *window) Canvas() core.Canvas {
    if gl := w.GLContext(); gl != nil {
        return glcanvas.New(gl)
    }
    return software.New(w.buffer)
}
```

---

## Phase 12: Wayland Platform (2 weeks)

### Goal
Fourth platform, modern Linux.

### Deliverables

**`platform/wayland/`** - ~1000 lines
- Registry binding
- XDG shell
- Shared memory buffers
- Input handling

### Reuse Validation
Zero changes to any code outside `platform/wayland/`.

---

## Phase 13: Android Platform (3 weeks)

### Goal
Mobile platform.

### Approach
Use Android NDK Native Activity. Minimal Java shim for launching.

### Deliverables

**`platform/android/`** - ~1000 lines
- Native Activity callbacks
- ANativeWindow handling  
- Touch input translation
- OpenGL ES context

**`cmd/android-bootstrap/`** - Minimal Java launcher

---

## Phase 14: Advanced Widgets (3 weeks)

### Goal
Production-ready widget set.

### Deliverables
Building on Phase 7 patterns:

**`widgets/list.go`** - Virtualized list (~150 lines)
**`widgets/tree.go`** - Tree view (~200 lines)
**`widgets/tabs.go`** - Tab container (~100 lines)
**`widgets/dialog.go`** - Modal dialogs (~100 lines)
**`widgets/menu.go`** - Menus and popups (~200 lines)
**`widgets/slider.go`** - Range input (~80 lines)
**`widgets/progress.go`** - Progress indicator (~50 lines)

Each widget remains small because:
1. Layout is handled by layout engine
2. Focus/hover/press are composed behaviors
3. Events come through single dispatcher
4. Drawing uses shared Canvas

---

## Phase 15: Accessibility (2 weeks)

### Goal
Screen reader support.

### Approach
Build accessibility tree parallel to widget tree:
```go
type Accessible interface {
    AccessibleRole() Role
    AccessibleName() string
    AccessibleValue() string
    AccessibleActions() []Action
}
```

Platform packages expose this to native accessibility APIs.

### Deliverables

**`a11y/tree.go`** - Accessibility tree (~150 lines)
**`platform/*/accessibility.go`** - Platform bridges (~200 lines each)

---

## Phase 16: Testing & Polish (2 weeks)

### Deliverables

**`testing/`** - Test utilities
- Headless platform for unit tests
- Widget test helpers
- Visual snapshot testing

**Examples/**
- Calculator app
- Todo app
- File browser

**Documentation**
- API docs
- Porting guide
- Widget development guide

---

## Code Budget Summary

| Package | Lines | Notes |
|---------|-------|-------|
| core/ | 600 | Types used everywhere |
| platform/shared/ | 400 | Reused by all platforms |
| platform/x11/ | 800 | X11-specific |
| platform/win32/ | 800 | Windows-specific |
| platform/darwin/ | 1200 | macOS-specific |
| platform/wayland/ | 1000 | Wayland-specific |
| platform/android/ | 1000 | Android-specific |
| render/software/ | 1200 | Reference renderer |
| render/gl/ | 800 | GPU renderer |
| text/ | 1200 | Font/text handling |
| layout/ | 600 | Pure math |
| widget/ | 500 | Shared behaviors |
| widgets/ | 1500 | All widgets combined |
| ui/ | 400 | Dispatch, focus |
| a11y/ | 500 | Accessibility |
| **Total** | **~12,000** | Complete toolkit |

### Comparison
- Without reuse architecture: 30,000+ lines (each widget 200-500 lines, duplicate platform logic)
- With reuse architecture: ~12,000 lines (widgets 40-100 lines, shared everything)

---

## Key Reuse Patterns

### 1. Platform drivers implement minimal primitives
Platforms don't contain event loop logic, dispatch logic, or rendering logic. They translate between OS and our types.

### 2. Widgets compose behaviors
`Focusable`, `Hoverable`, `Pressable` are embedded structs, not inheritance.

### 3. Single Canvas interface for all renderers
Software and GPU share exact same interface. Widget code is renderer-agnostic.

### 4. Layout engine is pure math
No Widget imports in layout package. Works on `Measurable` and `Bounds` interfaces.

### 5. Event dispatcher handles all routing
Widgets implement `Event(e) bool`, not `OnPointerDown`, `OnPointerUp`, etc.

### 6. Shared test infrastructure
Headless platform enables testing without display server.

---

## Timeline

| Phase | Duration | Running Total |
|-------|----------|---------------|
| 0: Core Types | 1 week | 1 week |
| 1: Platform Abstraction | 2 weeks | 3 weeks |
| 2: Software Renderer | 2 weeks | 5 weeks |
| 3: X11 Platform | 2 weeks | 7 weeks |
| 4: Event Dispatch | 1 week | 8 weeks |
| 5: Widget Core | 2 weeks | 10 weeks |
| 6: Layout Engine | 2 weeks | 12 weeks |
| 7: Basic Widgets | 2 weeks | 14 weeks |
| 8: Windows Platform | 2 weeks | 16 weeks |
| 9: macOS Platform | 3 weeks | 19 weeks |
| 10: Text Rendering | 3 weeks | 22 weeks |
| 11: GPU Backend | 3 weeks | 25 weeks |
| 12: Wayland Platform | 2 weeks | 27 weeks |
| 13: Android Platform | 3 weeks | 30 weeks |
| 14: Advanced Widgets | 3 weeks | 33 weeks |
| 15: Accessibility | 2 weeks | 35 weeks |
| 16: Testing & Polish | 2 weeks | 37 weeks |

**Total: ~9 months** (vs 15-18 months without reuse focus)

---

## Validation Gates

After each phase, verify:

1. **Phase 3 (X11)**: Demo runs, events work
2. **Phase 7 (Widgets)**: Basic app buildable
3. **Phase 8 (Windows)**: Same app runs unchanged
4. **Phase 9 (macOS)**: Same app runs unchanged
5. **Phase 12 (Wayland)**: Same app runs unchanged
6. **Phase 13 (Android)**: Same app runs with touch
7. **Phase 16**: All examples pass, docs complete

If platform N requires changes to widget/layout/render code, the abstraction is wrong. Fix it before proceeding.
