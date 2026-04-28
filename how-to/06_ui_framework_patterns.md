# How-To: UI Framework Patterns (SWT / JFace / Draw2D)
**Product:** FluoView / HPF / VPA
**Last Updated:** April 2026
**Source:** Distilled from US-62077 (empty CoolBar fix) — these are the patterns that were painful to discover and worth codifying.

---

## Stack at a glance

- **Java 21**, **Eclipse RCP 4.x (E4)**, OSGi
- **SWT** for widgets (single-threaded — display-thread-only for widget operations)
- **JFace** for viewers
- **Draw2D** for some custom controls (e.g., `FVCommonDraw2dControl`, spinners)
- **EMF/ECORE** for domain models that drive the UI

---

## Widget Hierarchy in an Image Tab

```
ImageView (E4 ViewPart)
  └─ FVCommonCoolBar (custom wrapper — contains GridLayout base)
       └─ SWT CoolBar (HORIZONTAL orientation)
            ├─ CoolItem[0] → PlayerComposite (channel/Z/Lambda controls for image 0)
            ├─ CoolItem[1] → MATLPlayerComposite (T-axis playback for image 0, if applicable)
            ├─ CoolItem[2] → PlayerComposite for image 1 (map image)
            └─ ...
       └─ ContentViewer (actual image display area, via createContentViewer)
```

---

## Key Classes and Their Roles

| Class | Location | Role |
|-------|----------|------|
| `ContentView` | `h-pf/.../ui.content` | Base view. Owns `FVCommonCoolBar coolbar` as **private** field. Provides `addCoolItem()` / `removeCoolItem()`. |
| `FVCommonCoolBar` | `h-pf/.../ui.commonparts` | Wraps SWT CoolBar. `addItemComposite()` re-parents, creates new CoolItem, sets size. **No idempotency** — every call creates a new CoolItem. `paintControl` draws a drag-handle thumb for **every** CoolItem. |
| `ImageContentView` | `fv/.../ui.image.viewpart` | Extends `ContentView`. Orchestrates player creation. |
| `PlayerComposite` | `fv/.../ui.image.player.parts` | Base player. Renders channel/Z/Lambda controls. Works correctly for all image types. |
| `AxisComposite` | `fv/.../ui.image.player.parts` | Renders axis controls. Iterates `player.getUiAxisParamList()`. |
| `MATLPlayerComposite` | `fv/.../protocol.matl.ui.image.viewpart.player` | Overrides `createAxisComposite()` to use `MATLAxisComposite`. |
| `MATLAxisComposite` | Same folder | Overrides `createPart()` for T-axis special handling. Renders zero controls for images without T-axis. |
| `MatlImageView` | `fv/.../protocol.matl.ui.image.viewpart` | Extends `ImageContentView`. MATL-specific overrides. |

---

## Player Creation Lifecycle

When `setFluoViewModel()` is called with a composite model, the base class iterates each `HPFImage`:

```
setFluoViewModel(model)
  └─ for each HPFImage in compositeModel:
       ├─ createUiPlayer(image)                        → player (e.g., MATLUiImagePlayer)
       ├─ getCreatedNewPlayerComposite(image, viewer, player)
       │    ├─ isCreatePlayerComposite(image, player)  → boolean: create player?
       │    ├─ createPlayerComposite(viewer, player)   → PlayerComposite or subclass
       │    └─ addCoolItem(composite)                  → wraps in CoolItem
       └─ setupView(image)
```

Hook points you can override in a subclass:

| Hook | Use it to... |
|------|--------------|
| `createUiPlayer` | Choose player implementation (`UiChartPlayer` / `UiImagePlayer` / `MATLUiImagePlayer`) |
| `isCreatePlayerComposite` | Suppress player for certain image types |
| `createPlayerComposite` | Choose composite type (base / MATL / custom) |
| `addCoolItem` | Control CoolItem placement (suppress, defer, reorder) |
| `setupView` | Post-setup hook |

---

## MATL Composite Model

Each MATL image tab's `CompositeHPFModel` contains **two images**:

| Image | Axes | Purpose |
|-------|------|---------|
| **Area image** | Depends on acquisition mode (see below) | The acquisition image |
| **Map image** | None — static reference | Overview scan (always no playback axes) |

Area image axis type by acquisition mode:

| Mode | Axis | Notes |
|------|------|-------|
| Cycle (multi-pass) | `TIMELAPSE` | Grows as cycles complete; `defaultMax` starts at 1 |
| Lambda scan | `LAMBDA` | `defaultMax > 1` at creation (wavelengths known upfront) |
| Z-stack | `ZSTACK` | `defaultMax > 1` at creation (slices known upfront) |
| Lambda excitation | `LAMBDAEX` | Same as Lambda |

**Test for axis presence** (returns `null` if not present):
```java
UiAxisParam tParam = player.getUiAxisParam(AxisType.TIMELAPSE);
if (tParam == null) {
    // image has no T-axis — skip MATL-specific T-axis composite
}
```

---

## SWT Gotchas (learned the hard way)

### Threading

- **SWT is single-threaded.** All widget operations must run on the display thread.
- Use `SWTHelper.execSyncInDisplayThread(control, Runnable)` to marshal.
- `synchronized` on widget code is a code smell — implies a contract SWT forbids.
- Inside an `execSyncInDisplayThread` block, widgets you entered with are still valid. Defensive disposed-checks outside this guarantee are noise.

### CoolBar / CoolItem

- `FVCommonCoolBar.addItemComposite()` **always appends** — never idempotent.
- To insert at a specific index, use `new CoolItem(coolbar, SWT.NONE, index)` constructor directly.
- `FVCommonCoolBar.paintControl` draws a drag-handle thumb (dotted line, 3px × 7px offset) for **every** CoolItem — even zero-size ones produce visible artifacts.
- The CoolBar is a **private** field of `ContentView`. No accessor. To get a reference from a subclass, walk up the widget hierarchy: `contentViewer.getParent().getParent()...` checking `instanceof CoolBar`.

### Widget Identity

- SWT widgets do not override `Object.equals()`. `.equals()` is identity-compare anyway.
- Use `==` for widget identity — matches codebase convention (e.g., `item.getControl() == deleteComposite` in `FVCommonCoolBar.removeItemComposite`).

### Parent-setting

- `new Composite(parent, style)` immediately parents to `parent`.
- `composite.setParent(newParent)` reparents an existing widget.
- A widget always has a parent at construction — `ensureX()`-style checks that expect no parent will always fail.

### Layout Propagation

Size/visibility changes on nested composites do **not** auto-propagate. Walk up:
```java
Composite p = changedWidget.getParent();
while (p != null && !p.isDisposed()) {
    p.layout(true, true);
    p = p.getParent();
}
```

---

## Player / Axis Model

### `UiAxisParam` events (`EventType`)

| Event | Fires when |
|-------|-----------|
| `POSITION` | Slider position changed |
| `DEFAULT_MAX` | Max value updated (e.g., new cycle data arrives) |
| `MAX` | Hard max changed |

Register listeners via `uiAxisParam.addParamChangeListener(ParamChangeListener)`.

### `AxisType` enum

- `TIMELAPSE` — T-axis (cycles, time-series)
- `LAMBDA` — wavelength
- `LAMBDAEX` — excitation wavelength
- `ZSTACK` — Z-depth slices

---

## Common Pitfalls

1. **Creating MATL-specific composites for non-MATL-axis images** — US-62077 root cause. Always check axis presence with `player.getUiAxisParam(AxisType.XXX)` before creating specialized composites.

2. **Calling `super.addCoolItem()` to show a deferred item** — it appends at the end, breaking the original order. Use indexed `CoolItem` constructor instead.

3. **Commenting out layout code "to see if it's needed"** — it usually IS needed. If you do it, add a TODO and verify by actually testing visibility changes.

4. **Overwriting Japanese comments with English translations** — convention is to **preserve** the Japanese comment and add English alongside for new code.

5. **Assuming widgets can be disposed mid-dispatch** — inside an `execSyncInDisplayThread` block, widgets you entered with are still valid.

6. **Hiding an empty CoolItem instead of removing it** — `paintControl` still draws the drag-handle thumb. Remove the CoolItem entirely.

---

## Quick Reference — Files you will likely read

- `MatlImageView.java` — MATL image tab controller (US-62077 fix lives here)
- `ImageContentView.java` — base for all image views; player lifecycle
- `ContentView.java` — base view with `addCoolItem`/`removeCoolItem`
- `FVCommonCoolBar.java` — CoolBar wrapper
- `MATLAxisComposite.java` — MATL-specific axis rendering
- `MATLPlayerComposite.java` — MATL player (very thin — just swaps AxisComposite)
- `PlayerComposite.java` — base player
- `AxisComposite.java` — base axis renderer

---

## Cross-references

- **Glossary**: `reference/01_domain_glossary.md` ▸ "UI Framework Terms" section
- **FAQ**: `onboarding/04_faq.md` ▸ "UI / SWT" section
- **FunctionEnabler integration**: `how-to/05_function_enabler_feature.md`
- **Class diagram**: `diagrams/04_class.md` (LSM image hierarchy)
- **Worked example**: `C:\home\claude\US-62077\sessions\US-62077-coolbar-fix.md`
