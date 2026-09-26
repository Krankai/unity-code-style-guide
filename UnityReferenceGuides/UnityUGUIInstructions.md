# Unity uGUI (Canvas) Best Practices

> **General Unity best practice.** Applies to any Unity 6 project. Change these only if you
> know why. Personal style preferences live in [`UnityStyleGuide.md`](../UnityStyleGuide.md);
> project-specific settings live in [`UnityCustomInstructions/`](../UnityCustomInstructions/).

Conventions and pitfalls for Canvas-based UI — **this project's UI system** for all runtime UI, per
[`UnityTechStack.md`](../UnityCustomInstructions/UnityTechStack.md). UI Toolkit is secondary: use
[UnityUIToolkitInstructions.md](UnityUIToolkitInstructions.md) only if a project switches to it, or for Editor
tooling. The two systems share almost no concepts, and advice from one is usually wrong for the other.

Most uGUI performance problems come from the same place: **something dirtied a Canvas, and Unity had
to rebuild it.** Almost every rule below is a way of rebuilding less often, or rebuilding less.

Table of contents:
- [uGUI or UI Toolkit?](#ugui-or-ui-toolkit)
- [Naming & organization](#naming--organization)
- [Canvas structure](#canvas-structure)
- [The rebuild pipeline](#the-rebuild-pipeline)
- [Performance rules](#performance-rules)
  - [Raycast targets](#raycast-targets)
  - [Layout groups](#layout-groups)
  - [Never animate UI with an Animator](#never-animate-ui-with-an-animator)
  - [Showing and hiding](#showing-and-hiding)
  - [Masking](#masking)
  - [Text](#text)
  - [Lists and pooling](#lists-and-pooling)
  - [Overdraw](#overdraw)
- [Code conventions for UI scripts](#code-conventions-for-ui-scripts)
- [Profiling uGUI](#profiling-ugui)
- [Troubleshooting](#troubleshooting)
- [Review checklist](#review-checklist)
- [Learn more](#learn-more)

---

## uGUI or UI Toolkit?

On this project the answer is **uGUI**, for every runtime screen — screen-space HUD and menus as well as world-space
UI. The table below is for a project deciding whether to switch; it is not a per-screen choice here.

| uGUI is the stronger fit when | UI Toolkit is the stronger fit when |
|---|---|
| UI lives in world space (health bars, diegetic screens) | Styling and reuse matter more than per-element effects (USS) |
| You need per-element shaders, materials, or particle effects | You're building Editor tooling as well |
| You depend on an asset-store UI package built on Canvas | You want layout that scales without manual anchoring |
| The team already knows RectTransform well | The team already knows CSS-style layout |

- ✅ UI Toolkit remains fine for **Editor** tooling (custom inspectors, windows).
- ❌ Don't add UI Toolkit runtime screens alongside uGUI on this project. On a project that does use both, never mix
  them for the same screen.

---

## Naming & organization

- ✅ PascalCase for UI GameObject names, matching the [style guide](../UnityStyleGuide.md#naming-files-and-folders):
  `StartButton`, `HealthBar`, `InventoryPanel`.
- ✅ Name by role, not by widget type. `StartButton` beats `Button (1)`; `HealthBarFill` beats `Image`.
- ✅ One prefab per screen or self-contained panel. Screens should be openable in isolation.
- ✅ Group folders by purpose: `Assets/UI/Screens/`, `Assets/UI/Components/`, `Assets/UI/Sprites/`.
- ✅ Give every interactive element a stable name — you will be looking for it in a hierarchy of
  eighty objects at 2am.
- ❌ Don't leave Unity's default names (`Image`, `Text (TMP)`, `Panel`) in a committed prefab.

```
Assets/UI/
├── Screens/
│   ├── MainMenu.prefab
│   ├── PauseMenu.prefab
│   └── InventoryScreen.prefab
├── Components/
│   ├── HealthBar.prefab
│   └── ItemSlot.prefab
└── Sprites/
```

---

## Canvas structure

A Canvas is the unit of batching **and** the unit of rebuilding. That single fact drives the whole
structure.

- ✅ **Split canvases by update frequency, not by visual nesting.** Static chrome on one Canvas,
  periodically-changing elements on another, per-frame elements on a third.
- ✅ A nested Canvas (a "sub-canvas") isolates its children: a dirty child no longer forces the parent
  to rebuild, and vice versa.
- ⚠️ **Don't over-split.** Batching does not cross a Canvas boundary. Every sub-canvas is its own set
  of draw calls. Three well-chosen canvases beat thirty.
- ⚠️ Toggling a nested Canvas's contents with `SetActive` while the parent Canvas holds many elements has been
  observed to cost disproportionately, for reasons not established. Profile it if you rely on that pattern.
- ✅ Keep elements that batch together on the same Canvas: same material, same texture (ideally one
  atlas), same Z.
- ⚠️ An `Image`/`RawImage` with no sprite renders with Unity's built-in white texture (`UnityWhite` in the Frame
  Debugger). It isn't in your atlas, so it breaks the batch. Put a small (e.g. 4×4) white sprite in the atlas and use
  it for plain rectangles.
- ❌ Never put a whole game's UI on one Canvas. Changing one label re-analyses every element on it.
- ✅ Set **Pixel Perfect** off unless you need it — it forces extra layout work.
- ⚠️ World Space canvases need an explicit **Event Camera** assigned, or raycasts silently do nothing.
  This is the single most common "my world-space button doesn't work" cause.

```
HUDCanvas            (Screen Space - Overlay)   ← static frame, never rebuilds
├── Background
├── Borders
│
├── StatsCanvas      (nested Canvas)            ← rebuilds a few times a second
│   ├── HealthBar
│   └── AmmoCounter
│
└── TimerCanvas      (nested Canvas)            ← rebuilds every frame
    └── TimerLabel
```

**Render mode:**

| Mode | Use for | Watch out for |
|---|---|---|
| Screen Space – Overlay | Most HUD and menus | Ignores cameras entirely — no post-processing, no render features |
| Screen Space – Camera | UI that needs post-processing or to sit between scene layers | Needs a camera reference; sensitive to plane distance |
| World Space | Diegetic UI, health bars above units | Needs an Event Camera; scales with distance |

---

## The rebuild pipeline

Understanding these two separate passes explains most uGUI performance advice:

1. **Layout rebuild** — recalculates RectTransform positions and sizes. Triggered by any
   `LayoutGroup`, `ContentSizeFitter`, or a `SetDirty` on a layout element.
2. **Graphic rebuild** — regenerates the vertex mesh and re-batches. Triggered by changing a colour,
   sprite, text, or anything else visual.

Both run in `Canvas.SendWillRenderCanvases` at the end of the frame, **not** at the moment you set
the property. Setting `text` five times in one frame costs one rebuild, not five — but setting it
every frame costs one rebuild every frame.

- ❌ Never call `Canvas.ForceUpdateCanvases()`. It forces the rebuild to happen immediately and
  synchronously, and is almost always covering for an ordering bug elsewhere.
- ✅ If you need a layout value immediately after changing it, call
  `LayoutRebuilder.ForceRebuildLayoutImmediate(rect)` on **that one RectTransform** rather than
  forcing every canvas in the scene.

---

## Performance rules

### Raycast targets

- ✅ **Turn `Raycast Target` off on every non-interactive Graphic.** It's on by default on every
  `Image` and `TextMeshProUGUI` you create.
- ℹ️ Every pointer event walks the list of raycast targets and runs a hit test against each one. A
  screen with 200 decorative images does 200 hit tests per pointer, per frame.
- ✅ This is the cheapest, highest-yield fix in uGUI and nearly every project has it wrong.
- ✅ Make it the project default instead of unticking it per instance: create `Image`, `RawImage` and
  `TextMeshProUGUI` Presets with Raycast Target off and set them as defaults in the Preset Manager.
- ✅ For an invisible input blocker or drag area, don't use a plain transparent `Image` — it still generates
  geometry and still draws. Use a raycast target that draws nothing. Which one depends on the Unity version (check
  [`UnityTechStack.md`](../UnityCustomInstructions/UnityTechStack.md)):

| Unity version | Use | Notes |
|---|---|---|
| **6.5 and later** (uGUI 2.5+) | Unity's built-in **`RaycastReceiver`** component | Official; no draw calls. Has **Raycast Target** and **Raycast Padding**. Don't install a third-party equivalent. |
| **6.0 – 6.4** | [Unity-NonDrawingGraphic](https://github.com/IvanMurzak/Unity-NonDrawingGraphic)'s `NonDrawingGraphic` (optional package — OpenUPM `extensions.unity.nondrawinggraphic`, MIT) | A `Graphic` that builds no mesh. Add it to a GameObject under a Canvas. |
| **6.0 – 6.4**, no package | The `NonDrawingGraphic` class below | Same idea, written by hand. Don't keep it alongside the package — they share the class name. |

- ℹ️ A no-code fallback on any version: a transparent `Image` with **Cull Transparent Mesh** ticked on its
  `CanvasRenderer`, which lets Unity skip a mesh whose vertex alpha is near zero. Its mesh is still built before
  being skipped, so it costs slightly more than the options above. That it still receives clicks is expected (uGUI's
  hit test ignores alpha by default) but not yet confirmed in a project.

```csharp
// Unity 6.0 - 6.4 fallback when the Unity-NonDrawingGraphic package isn't installed
/// <summary>
/// An invisible, zero-geometry raycast target. Use instead of a transparent Image
/// for full-screen input blockers and drag areas.
/// </summary>
public class NonDrawingGraphic : Graphic
{
    public override void SetMaterialDirty() { }
    public override void SetVerticesDirty() { }

    protected override void OnPopulateMesh(VertexHelper vh)
    {
        vh.Clear();
    }
}
```

### Layout groups

- ⚠️ `HorizontalLayoutGroup`, `VerticalLayoutGroup`, `GridLayoutGroup` and `ContentSizeFitter` are the
  most expensive components in uGUI. Each one adds a layout pass over its children.
- ❌ **Never nest layout groups.** A `ContentSizeFitter` inside a `VerticalLayoutGroup` inside another
  `VerticalLayoutGroup` triggers a cascade of rebuilds — cost grows multiplicatively with depth.
- ✅ Use layout groups for content whose size you genuinely can't know ahead of time (localized text,
  dynamic lists).
- ✅ For layouts that are fixed at design time, **bake the result**: set the anchors and sizes once in
  the Editor and delete the layout group. Anchoring does the same job for free.
- ✅ If a layout only changes when the player does something, disable the group component after the
  first rebuild and re-enable it when the content actually changes.
- ✅ Remove a layout group that was only added for convenience while authoring a prefab before you save it. If an
  element just needs to stay relative to its parent, that's an anchoring problem, not a layout-group one.

```csharp
// Static list built once - let the layout group run, then get out of the way
private void Start()
{
    PopulateSlots();
    LayoutRebuilder.ForceRebuildLayoutImmediate(_contentRect);
    _layoutGroup.enabled = false;    // No further layout passes
}
```

### Never animate UI with an Animator

- ❌ An `Animator` on a UI element **dirties its graphics every frame it is enabled**, even when every
  animated value is identical to last frame. Animators have no no-op check.
- ✅ Drive UI transitions from code, a tween library, or `Button`'s built-in Color/Sprite transition.
- ✅ If you must use an Animator, disable the component when the animation finishes.
- ⚠️ This applies to Unity's default Button "Animation" transition mode too. Prefer **Color Tint** or
  **Sprite Swap**.

```csharp
// Simple fade that doesn't dirty layout at all. Pass this.GetCancellationTokenOnDestroy() as the token:
// destroying the object cancels the loop, so no "this == null" checks are needed
private async UniTask TaskFadeIn(CanvasGroup group, float duration, CancellationToken token)
{
    try
    {
        float elapsed = 0f;

        while (elapsed < duration)
        {
            elapsed += Time.unscaledDeltaTime;
            group.alpha = Mathf.Clamp01(elapsed / duration);
            await UniTask.NextFrame(token);
        }

        group.alpha = 1f;
    }
    catch (System.OperationCanceledException)
    {
        throw;
    }
    catch (System.Exception e)
    {
        AppLogger.LogException(e);
    }
}
```

### Showing and hiding

- ✅ **Default: hide with a `CanvasGroup`.** Set `alpha = 0`, `interactable = false` and `blocksRaycasts = false` —
  alpha alone doesn't block input. It is by far the cheapest toggle (see the table), and the same component does
  fades.
- ⚠️ A `CanvasGroup` leaves the GameObject active: `Update`, running UniTask loops and R3 subscriptions keep going
  while the screen is invisible. If a hidden screen runs **heavy per-frame logic**, use `SetActive(false)` for that
  screen instead — the expensive toggle, but the only one that stops the per-frame work.
- ❌ Don't `SetActive` a screen that is toggled often for any other reason. Re-enabling forces a full layout and
  graphic rebuild, which is exactly the hitch you see when opening a menu.
- ℹ️ Disabling the `Canvas` component sits in between: it stops rendering and raycasts for that whole Canvas, keeps its
  batch, and leaves scripts running.

| Technique | Toggle cost, show / hide ¹ | Stops raycasts | Stops scripts | Use for |
|---|---|---|---|---|
| `CanvasGroup` (alpha 0, not interactable, no raycasts) | 3.64 / 3.40 ms | ✅ with `blocksRaycasts = false` | ❌ | **Default** — any show/hide, and fades |
| `Canvas.enabled = false` | 61.25 / 61.23 ms | ✅ | ❌ | Hiding a whole Canvas when a `CanvasGroup` isn't set up |
| `SetActive(false)` | 323.79 / 209.93 ms | ✅ | ✅ | A hidden screen whose per-frame logic must stop |

¹ From the handoff source material: 1280 `Image` objects, measured in the Editor without Deep Profile. Read the
ordering, not the absolute numbers. It measures the toggle only, not what a hidden screen costs per frame — profile
large screens that stay hidden.

```csharp
[SerializeField] private CanvasGroup _canvasGroup;

public void SetVisible(bool isVisible)
{
    _canvasGroup.alpha = isVisible ? 1f : 0f;
    _canvasGroup.interactable = isVisible;      // alpha alone doesn't block input
    _canvasGroup.blocksRaycasts = isVisible;
}
```

### Masking

- ✅ Prefer **`RectMask2D`** over `Mask`. It clips by rectangle in the shader — no stencil buffer, no
  extra draw calls, and it doesn't break batching the way `Mask` does.
- ⚠️ Use `Mask` only when you genuinely need a non-rectangular mask from a sprite's alpha.
- ⚠️ Every `Mask` costs two extra draw calls (write stencil, clear stencil) *per mask*.
- ⚠️ `RectMask2D` isn't free: an enabled one pays a CPU culling cost every frame, proportional to how many
  graphics it masks, even when nothing moves. Keep the masked count small, and set `enabled = false` on a
  `RectMask2D` while masking isn't needed.

### Text

- ✅ Use **TextMeshPro** (`TextMeshProUGUI`), not the legacy `Text` component.
- ❌ Turn **Auto Size / Best Fit** off. It binary-searches for a font size on every rebuild.
- ✅ **Set numbers and formatted values with `SetText`**, not `.text = value.ToString()`. `ToString()` allocates a new
  string on every call; `SetText("{0}", value)` formats into TextMeshPro's own buffer instead.
- ⚠️ Assigning `.text` or calling `SetText` dirties the mesh even if the result is identical. Guard it:

```csharp
private int _lastScore = -1;

private void UpdateScore(int score)
{
    if (score == _lastScore) return;    // Skip the rebuild entirely

    _lastScore = score;
    _scoreLabel.SetText("{0}", score);  // No string allocated
}
```

- ✅ For multi-value or heavier formatting, [ZString](https://github.com/Cysharp/ZString) (optional, see
  [`UnityTechStack.md`](../UnityCustomInstructions/UnityTechStack.md)) writes straight into the label:
  `_positionLabel.SetTextFormat("Position: {0}, {1}", x, y);` (`using Cysharp.Text;`).
- ⚠️ ZString enables its TextMeshPro support automatically only when `com.unity.textmeshpro` is in the package
  manifest. Unity 6 ships TextMeshPro inside the uGUI package, so if `SetTextFormat` isn't found, add
  `ZSTRING_TEXTMESHPRO_SUPPORT` to Scripting Define Symbols. (Not yet confirmed in a project.)
- ℹ️ For font atlas and fallback configuration, see the TextMeshPro section of the uGUI package documentation
  (under [Learn more](#learn-more)).

### Lists and pooling

- ❌ Never `Instantiate`/`Destroy` list rows as the player scrolls. It allocates, triggers a layout
  rebuild, and causes GC spikes.
- ✅ Pool row prefabs with `UnityEngine.Pool.ObjectPool<T>` — see
  [Object Pooling](UnityDesignPatternsInstructions.md#object-pooling).
- ✅ For long lists, recycle a fixed number of rows and rebind their data as they move offscreen.
  Twenty rows can present ten thousand items. If EnhancedScroller is installed (optional, see
  [`UnityTechStack.md`](../UnityCustomInstructions/UnityTechStack.md)), use it rather than a hand-rolled recycler.
- ✅ Populate a list while it's hidden, then show it — one rebuild instead of one per row.

### Overdraw

- ⚠️ uGUI draws back-to-front with no depth rejection. Every stacked full-screen panel is a full
  screen of transparent fill, and fill rate is what kills mobile.
- ✅ When a full-screen menu covers the HUD, disable the HUD Canvas rather than leaving it behind.
- ✅ Use the **Overdraw** draw mode in the Scene view to find the hot spots.
- ✅ Avoid large, mostly-transparent images. Nine-slice a small sprite instead.

---

## Code conventions for UI scripts

These follow the same rules as the rest of the codebase — see the
[style guide](../UnityStyleGuide.md) — with a few uGUI specifics:

- ✅ Cache component references in `Awake()`. Never `GetComponentInChildren` per frame.
- ✅ Consume `Button.onClick` through R3: `_button.OnClickAsObservable().Subscribe(...).AddTo(this)` in `Start`. It
  subscribes once per instance, so a pooled row that is reused can't stack listeners, and it ends when the row is
  destroyed. Don't `AddListener` from code — see [Events](../UnityStyleGuide.md#events).
- ✅ Expose the View's own events as `Observable<T>` over a private `Subject<T>`, named `On` + past tense.
- ✅ Keep UI scripts presentational — they are the **View** in
  [MVC](UnityArchitectureInstructions.md#mvc-layering). They render state and forward input to the Controller; they
  don't own game logic.
- ❌ Don't reach across the hierarchy with `transform.Find("Panel/Row/Label")`. Serialize the
  reference.

```csharp
[RequireComponent(typeof(Button))]
public class ItemSlotView : MonoBehaviour
{
    [SerializeField] private Image _icon;
    [SerializeField] private TextMeshProUGUI _countLabel;

    private readonly Subject<ItemConfig> _clicked = new();
    private Button _button;
    private ItemConfig _item;

    public Observable<ItemConfig> OnClicked => _clicked;

    private void Awake()
    {
        _button = GetComponent<Button>();
    }

    private void Start()
    {
        // Once per instance - a pooled row that is reused keeps this single subscription
        _button.OnClickAsObservable().Subscribe(HandleClicked).AddTo(this);
    }

    private void OnDestroy()
    {
        _clicked.Dispose();
    }

    public void Bind(ItemConfig item, int count)
    {
        _item = item;
        _icon.sprite = item.Icon;
        _countLabel.SetText("{0}", count);
    }

    private void HandleClicked(Unit _)
    {
        _clicked.OnNext(_item);
    }
}
```

---

## Profiling uGUI

Open the Profiler and look for these markers under **PlayerLoop → PostLateUpdate**:

| Marker | Means | Usual cause |
|---|---|---|
| `Canvas.SendWillRenderCanvases` | Something dirtied a canvas this frame | Text set every frame, Animator on UI |
| `CanvasUpdateRegistry.PerformUpdate` | Layout and graphic rebuild pass | Layout groups, ContentSizeFitter |
| `Canvas.BuildBatch` | Re-batching geometry | Too much on one Canvas |
| `Graphic.Rebuild` | A specific graphic regenerated its mesh | `.text` or `.sprite` assignment |
| `IndexedSet.Sort` / `Canvas.SortDrawTrees` | Sorting draw order | Many canvases, or changing sibling order |

- ✅ The **UI** and **UI Details** Profiler modules show which canvases rebuilt and why — check the
  batch count and the "rebuild reason" column before you start optimizing blind.
- ✅ Profile on the target device. Canvas rebuild cost is CPU-bound and scales with device, and fill
  rate problems only show up on real mobile hardware.
- ℹ️ See [UnityPerformanceOptimizationInstructions.md](UnityPerformanceOptimizationInstructions.md#profiling-markers)
  for the general profiling workflow.

---

## Troubleshooting

**Button does nothing when clicked.**
Work down in order: is there an `EventSystem` in the scene at all; does the Canvas have a
`GraphicRaycaster`; is the Button's `Raycast Target` on; is something invisible on top of it (a
full-screen panel with `Raycast Target` left on is the usual culprit); is the Button `interactable`.

```csharp
// Diagnostic: what is actually under the pointer? (Input System, per the tech stack)
private void LogRaycastsUnderPointer()
{
    var data = new PointerEventData(EventSystem.current) { position = Pointer.current.position.ReadValue() };
    var results = new List<RaycastResult>();
    EventSystem.current.RaycastAll(data, results);

    AppLogger.Log($"{results.Count} raycast target(s) under pointer, front to back:");
    foreach (RaycastResult r in results)
    {
        AppLogger.Log($"  {r.gameObject.name}   (sortingOrder {r.sortingOrder})", r.gameObject);
    }
}
```

**World-space UI ignores clicks entirely.**
The Canvas has no **Event Camera** assigned. Screen-space canvases find one automatically; world
space does not, and the failure is silent.

**Layout collapses to zero size, or elements pile on top of each other.**
A `ContentSizeFitter` inside a `LayoutGroup` that is itself size-driven — each is waiting for the
other. Give one of them a fixed dimension.

**Opening a screen causes a visible hitch.**
`SetActive(true)` on a canvas subtree forces a full layout and graphic rebuild. Hide and show it with a
`CanvasGroup` instead — see [Showing and hiding](#showing-and-hiding).

**A hidden screen still costs frame time.**
It was hidden with a `CanvasGroup` (or by disabling its Canvas), so its scripts keep running. If that per-frame
work is heavy, hide that screen with `SetActive(false)` instead.

**A button click fires twice on a pooled row.**
Listeners were added each time the row was enabled or bound. Subscribe once in `Start` with
`OnClickAsObservable().Subscribe(...).AddTo(this)` — see [Code conventions](#code-conventions-for-ui-scripts).

**UI flickers or elements draw in the wrong order.**
Elements on one Canvas with different Z values, or sibling order changing at runtime. uGUI draws in
hierarchy order; Z fighting breaks batching and ordering both.

**Text is blurry in world space.**
Canvas `Dynamic Pixels Per Unit` is too low for the scale, or the RectTransform is scaled rather than
sized. Set the rect to the size you want and leave scale at 1.

**Profiler shows constant `Canvas.SendWillRenderCanvases` with nothing visibly changing.**
Something dirties the canvas every frame: an `Animator` on a UI element, or `.text` assigned
unconditionally in `Update`.

---

## Review checklist

When reviewing uGUI work, check these in order — roughly highest impact first:

| Check | Look for |
|---|---|
| Raycast targets | `Raycast Target` ticked on decorative images and labels; a transparent `Image` used as an input blocker instead of `RaycastReceiver` (6.5+) or `NonDrawingGraphic` |
| Canvas count | One giant Canvas, or thirty tiny ones |
| Layout groups | Nested groups, `ContentSizeFitter` inside a layout group |
| Animators | Any `Animator` component on a UI GameObject |
| Show/hide | `SetActive` used where a `CanvasGroup` would do (fine only to stop heavy per-frame logic); `CanvasGroup` hidden without `blocksRaycasts = false` |
| Masks | `Mask` used where `RectMask2D` would do; a `RectMask2D` left enabled over many graphics when not needed |
| Text | Auto Size enabled; `.text` assigned unconditionally each frame; `.text = x.ToString()` where `SetText` would do |
| Batching | `Image`/`RawImage` with no sprite (UnityWhite) on an atlased Canvas |
| Lists | `Instantiate` per row, no pooling |
| World space | Missing Event Camera |
| Code | `GetComponent` in `Update`; `onClick.AddListener` from code instead of `OnClickAsObservable()`; a View exposing `event Action` instead of `Observable<T>` |

---

## Learn more

- [Unity UI (uGUI) package documentation](https://docs.unity3d.com/Packages/com.unity.ugui@2.0/manual/index.html)
- [uGUI 2.5 visual components, incl. `RaycastReceiver`](https://docs.unity3d.com/Packages/com.unity.ugui@2.5/manual/UIVisualComponents.html) (Unity 6.5+)
- [Unity UI optimization tips](https://unity.com/how-to/unity-ui-optimization-tips)
- [Split canvas for dynamic objects](https://support.unity.com/hc/en-us/articles/115000355466-Split-canvas-for-dynamic-objects)
