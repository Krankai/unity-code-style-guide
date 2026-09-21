---
description: Guidelines for debugging Unity projects via MCP connection
applyTo: "**/*.cs"
---

# Unity Debugging Instructions for LLM Coding Tools

> **General Unity best practice.** Applies to any Unity 6 project. Change these only if you
> know why. Personal style preferences live in [`UnityStyleGuide.md`](../UnityStyleGuide.md);
> project-specific settings live in [`UnityCustomInstructions/`](../UnityCustomInstructions/).


> **Logging in this guide.** Diagnostic snippets call `Debug.Log` directly because they are temporary: add them,
> read the Console, remove them. Anything that stays in the codebase goes through `AppLogger`, whose `Log` and
> `LogWarning` are stripped unless `ENABLE_LOGS` is defined — see [Debugging](../UnityStyleGuide.md#debugging) in the
> style guide and [Conditional Logging](#diagnostic-code-patterns) below.

## Overview

This document provides instructions for AI coding assistants connected to Unity projects via MCP (Model Context Protocol).
Use these guidelines when helping developers debug Unity applications.

## Table of Contents

- [Diagnostic Priority Order](#diagnostic-priority-order)
- [Console Output Analysis](#console-output-analysis)
- [SerializeField and Inspector Debugging](#serializefield-and-inspector-debugging)
- [Script Execution Order Issues](#script-execution-order-issues)
- [Null Reference Debugging](#null-reference-debugging)
- [Input System Debugging](#input-system-debugging)
- [Physics Debugging](#physics-debugging)
- [Animation and Animator Debugging](#animation-and-animator-debugging)
- [uGUI Debugging](#ugui-debugging)
- [UI Toolkit Debugging](#ui-toolkit-debugging-unity-6) (secondary)
- [Audio Debugging](#audio-debugging)
- [Async and Coroutine Debugging](#async-and-coroutine-debugging)
- [Event System Debugging](#event-system-debugging)
- [ScriptableObject Runtime Issues](#scriptableobject-runtime-issues)
- [Transform and Hierarchy Issues](#transform-and-hierarchy-issues)
- [Performance Debugging](#performance-debugging)
- [Scene and Asset Debugging](#scene-and-asset-debugging)
- [Build and Platform-Specific Debugging](#build-and-platform-specific-debugging)
- [Collaborative Debugging with AI Tools](#collaborative-debugging-with-ai-tools)
- [LLM-Focused Triage Prompts](#llm-focused-triage-prompts)
- [Learn more](#learn-more)

## LLM-Focused Triage Prompts
- Paste the exact **error/warning text + stack trace** (include file path and line number).
- State **Unity version** (e.g., 6.3.x), **platform/build type** (Editor/Dev/Release), and **Enter Play Mode Options** (domain/scene reload on/off).
- Give **repro steps** and name the **scene/prefab** and **scripts** involved.
- For **Input/UI**, list the active **action map/control scheme** and the **Canvas/prefab names** (uGUI) or
  **UXML/USS asset names** (UI Toolkit) involved.
- Mention any **recent code or asset changes** just before the bug appeared.
- If possible, share a **minimal log snippet** around the failure (no screenshots).

## Diagnostic Priority Order

When investigating Unity issues, check these areas in order:

1. **Console errors and warnings** - Always start here
2. **Null reference exceptions** - Most common Unity issue
3. **Serialization state** - Inspector values vs runtime values
4. **Lifecycle timing** - Script execution order problems
5. **Scene/Prefab state** - Missing references, disabled objects
6. **Physics/Rendering settings** - Layer masks, culling, collision matrices

---

## Console Output Analysis

### Error Categories

| Prefix | Meaning | Typical Cause |
|--------|---------|---------------|
| `NullReferenceException` | Missing object reference | Unassigned SerializeField, destroyed object, wrong execution order |
| `MissingReferenceException` | Reference to destroyed object | Accessing object after Destroy() |
| `MissingComponentException` | GetComponent returned null | Component not attached, wrong type |
| `IndexOutOfRangeException` | Array/List bounds exceeded | Off-by-one errors, empty collections |
| `InvalidOperationException` | Operation not allowed | Modifying collection while iterating |

### Stack Trace Interpretation

```
NullReferenceException: Object reference not set to an instance of an object
  at PlayerMover.Update () [0x00012] in Assets/Scripts/PlayerMover.cs:47
  at UnityEngine.Internal.$MethodUtility.InvokeMethod (...)
```

**Key information:**
- Script name: `PlayerMover`
- Method: `Update()`
- Line number: `47`
- File path: `Assets/Scripts/PlayerMover.cs`

### Common Warning Patterns

```csharp
// Warning: "SendMessage cannot be called during Awake, CheckConsistency, or OnValidate"
// Fix: Defer to Start() or use Invoke/Coroutine

// Warning: "The referenced script on this Behaviour is missing!"
// Fix: Script file deleted or class name doesn't match filename

// Warning: "You are trying to create a MonoBehaviour using the 'new' keyword"
// Fix: Use AddComponent<T>() or Instantiate()
```

---

## SerializeField and Inspector Debugging

### Verify Serialization State

Four ways to declare the same reference, and what each does in the Inspector:

| Declaration | Serialized | In Inspector | Verdict |
|---|---|---|---|
| `[SerializeField] private GameObject _target;` | ✅ | ✅ visible | ✅ Correct |
| `private GameObject _target;` | ❌ | ❌ absent | The field is null at runtime and you can't assign it |
| `public GameObject Target;` | ✅ | ✅ visible | Works, but breaks encapsulation — avoid |
| `[HideInInspector] public GameObject Target;` | ✅ | ❌ hidden | Deliberate: persisted but not designer-editable |

```csharp
// ✅ The one you want
[SerializeField] private GameObject _target;
```

⚠️ A field that "won't show up in the Inspector" is almost always the second row — the
`[SerializeField]` attribute is missing. Unity gives no warning for this.

### Runtime vs Inspector Values

```csharp
// Debug serialized values at runtime
private void OnValidate()
{
    Debug.Log($"[Editor] _target assigned: {_target != null}");
}

private void Awake()
{
    Debug.Log($"[Runtime] _target assigned: {_target != null}");
}
```

### Check for Prefab Overrides

When a SerializeField appears assigned in the Prefab but null at runtime:
1. Check if the scene instance has an override (bold in Inspector)
2. Check if the value was cleared in a prefab variant
3. Check if OnValidate() or Reset() is clearing the value

---

## Script Execution Order Issues

### Unity Lifecycle Order

```
[Script Execution Order]
    ↓
Awake()           ← Object initialization, references to self
    ↓
OnEnable()        ← Subscribe to plain C# events (Unity, third-party, event Action)
    ↓
Start()           ← References to other objects; R3 subscriptions (.AddTo(this))
    ↓
FixedUpdate()     ← Physics updates (fixed timestep)
    ↓
Update()          ← Game logic (every frame)
    ↓
LateUpdate()      ← Camera follow, post-processing
    ↓
OnDisable()       ← Unsubscribe plain C# events
    ↓
OnDestroy()       ← Cleanup; dispose owned Subjects (R3 .AddTo(this) subscriptions end here)
```

### Diagnosing Order Problems

```csharp
// Add execution order logging
private void Awake()
{
    Debug.Log($"{GetType().Name}.Awake() on {gameObject.name}", this);
}

private void Start()
{
    Debug.Log($"{GetType().Name}.Start() on {gameObject.name}", this);
}
```

### Setting Script Execution Order

```csharp
// Force early execution
[DefaultExecutionOrder(-100)]
public class GameManager : MonoBehaviour { }

// Force late execution
[DefaultExecutionOrder(100)]
public class UIManager : MonoBehaviour { }
```

### Common Timing Issues

| Symptom | Likely Cause | Solution |
|---------|--------------|----------|
| Reference null in `Awake()` | Other object not yet initialized | Move to `Start()` |
| Reference null in `Start()` | Object created later in scene | Inject it through VContainer, or have the late object register itself with a service. Avoid `FindAnyObjectByType` at runtime |
| Injected service null, or VContainer can't resolve it in a scene | The scene's scope isn't parented to the boot scope | See [Parent link for the next scope](UnityScenesAndLifecycleInstructions.md#parent-link-for-the-next-scope) |
| Camera jitter | Camera in `Update()`, target in `Update()` | Move camera to `LateUpdate()` |
| Physics inconsistency | Physics logic in `Update()` | Move to `FixedUpdate()` |

- If state resets or Awake/OnEnable timing looks odd, verify **Enter Play Mode Options** (Project Settings → Editor). Domain reload off keeps static state; scene reload off keeps scene state.

---

## Null Reference Debugging

### Systematic Null Checking

```csharp
// Pattern: Validate all SerializeFields in Awake/Start
private void Awake()
{
    Debug.Assert(_playerTransform != null, "PlayerTransform not assigned!", this);
    Debug.Assert(_healthBar != null, "HealthBar not assigned!", this);
    Debug.Assert(_audioSource != null, "AudioSource not assigned!", this);
}

// Pattern: Null-conditional for optional references
_optionalComponent?.DoSomething();

// Pattern: Explicit null check with error
if (_requiredComponent == null)
{
    Debug.LogError($"Required component missing on {gameObject.name}", this);
    enabled = false;
    return;
}
```

### GetComponent Failure Patterns

```csharp
// Issue: Component on different GameObject
Rigidbody rb = GetComponent<Rigidbody>();  // Only checks THIS object

// Fix: Specify where to look
Rigidbody rb = GetComponentInChildren<Rigidbody>();
Rigidbody rb = GetComponentInParent<Rigidbody>();
Rigidbody rb = someOtherObject.GetComponent<Rigidbody>();

// Issue: Component added at runtime not yet available
// Fix: Use RequireComponent or check timing
[RequireComponent(typeof(Rigidbody))]
public class PhysicsBody : MonoBehaviour { }
```

### Destroyed Object Access

```csharp
// Issue: Accessing destroyed object
private GameObject _enemy;

private void Update()
{
    // This throws MissingReferenceException after enemy is destroyed
    float dist = Vector3.Distance(transform.position, _enemy.transform.position);
}

// Fix: Unity's fake null check
if (_enemy != null)   // Works for destroyed objects
{
    float dist = Vector3.Distance(transform.position, _enemy.transform.position);
}

// Note: C# null check doesn't catch destroyed objects
if (_enemy is not null)   // WRONG - doesn't detect destroyed Unity objects
```

---

## Input System Debugging

### Device Connection Issues

```csharp
// Debug: Monitor device changes
private void OnEnable()
{
    InputSystem.onDeviceChange += HandleDeviceChange;
}

private void OnDisable()
{
    InputSystem.onDeviceChange -= HandleDeviceChange;
}

private void HandleDeviceChange(InputDevice device, InputDeviceChange change)
{
    Debug.Log($"Device '{device.displayName}' {change}");
}

// Debug: List all connected devices
private void LogConnectedDevices()
{
    foreach (var device in InputSystem.devices)
    {
        Debug.Log($"Device: {device.displayName} ({device.GetType().Name})");
    }
}
```

### PlayerInput Component Issues

```csharp
// Issue: Actions not firing
// Check 1: Is PlayerInput component enabled?
// Check 2: Is the correct Action Map active?

// Debug: Log all action triggers (a C# event on PlayerInput - += / -= pairing applies)
private PlayerInput _playerInput;

private void Awake()
{
    _playerInput = GetComponent<PlayerInput>();
}

private void OnEnable()
{
    _playerInput.onActionTriggered += HandleActionTriggered;
}

private void OnDisable()
{
    _playerInput.onActionTriggered -= HandleActionTriggered;
}

private void HandleActionTriggered(InputAction.CallbackContext context)
{
    Debug.Log($"Action: {context.action.name}, Phase: {context.phase}");
}

// Issue: Wrong action map active
var playerInput = GetComponent<PlayerInput>();
Debug.Log($"Current Action Map: {playerInput.currentActionMap?.name ?? "None"}");

// Fix: Switch action map
playerInput.SwitchCurrentActionMap("Gameplay");
```

### Input Action Debugging

```csharp
// Debug: Check if action is bound
private void DebugAction(InputAction action)
{
    Debug.Log($"Action: {action.name}");
    Debug.Log($"  Enabled: {action.enabled}");
    Debug.Log($"  Bindings: {action.bindings.Count}");

    foreach (var binding in action.bindings)
    {
        Debug.Log($"    Path: {binding.effectivePath}");
    }
}

// Issue: Action enabled but not responding
// Check: Is the binding path correct for the current device?
// Check: Is another action consuming the input?

// Debug: Read action value directly
private void Update()
{
    var moveAction = _inputActions.Player.Move;
    Vector2 value = moveAction.ReadValue<Vector2>();
    Debug.Log($"Move: {value}, Phase: {moveAction.phase}");
}
```

### Common Input System Issues

| Symptom | Check | Solution |
|---------|-------|----------|
| No input response | Is InputActionAsset assigned? | Assign in Inspector or via code |
| Actions not firing | Is action map enabled? | Call `actionMap.Enable()` |
| Wrong device input | Check control scheme | Verify bindings for target device |
| Input works in Editor only | Is Input System package in build? | Check Player Settings → Active Input Handling |
| Duplicate input events | Multiple PlayerInput components? | Use single PlayerInput or manual action management |

- Use **Input Debugger** (Window → Analysis → Input Debugger) to inspect devices, events, and action states live. Verify the active control scheme matches the connected device.

---

## Physics Debugging

For the rules themselves — which callbacks fire for which Rigidbody combination, collider
limits, and tunnelling — see [Physics](UnityPhysicsInstructions.md).

### Layer and Collision Issues

```csharp
// Debug: Check layer configuration
Debug.Log($"Object layer: {gameObject.layer} ({LayerMask.LayerToName(gameObject.layer)})");

// Debug: Visualize raycast
Debug.DrawRay(origin, direction * maxDistance, Color.red, 2f);

// Debug: Check what layers a LayerMask includes
private void LogLayerMask(LayerMask mask)
{
    for (int i = 0; i < 32; i++)
    {
        if ((mask.value & (1 << i)) != 0)
        {
            Debug.Log($"Layer {i}: {LayerMask.LayerToName(i)}");
        }
    }
}
```

### Collision Matrix Verification

Check Edit → Project Settings → Physics (or Physics 2D):
- Verify layer collision matrix has expected checkboxes enabled
- Both objects must be on layers that collide with each other

### Rigidbody Issues

| Symptom | Check | Solution |
|---------|-------|----------|
| No collision detected | Is Rigidbody present? | Add Rigidbody to at least one object |
| Collision but no callback | Is trigger enabled incorrectly? | Match OnCollision vs OnTrigger methods |
| Objects pass through | Is one kinematic with no continuous detection? | Enable Continuous collision detection |
| Jittery movement | Moving in Update() | Move physics objects in FixedUpdate() |

### Trigger vs Collision Methods

```csharp
// Colliders (IsTrigger = false)
private void OnCollisionEnter(Collision collision) { }
private void OnCollisionStay(Collision collision) { }
private void OnCollisionExit(Collision collision) { }

// Triggers (IsTrigger = true)
private void OnTriggerEnter(Collider other) { }
private void OnTriggerStay(Collider other) { }
private void OnTriggerExit(Collider other) { }

// 2D variants
private void OnCollisionEnter2D(Collision2D collision) { }
private void OnTriggerEnter2D(Collider2D other) { }
```

---

## Animation and Animator Debugging

### Animator State Issues

```csharp
// Debug: Log current animator state
private void Update()
{
    var stateInfo = _animator.GetCurrentAnimatorStateInfo(0);
    Debug.Log($"State: {stateInfo.shortNameHash}, NormalizedTime: {stateInfo.normalizedTime}");
}

// Debug: Check if specific state is playing
private bool IsPlaying(string stateName)
{
    var stateInfo = _animator.GetCurrentAnimatorStateInfo(0);
    return stateInfo.IsName(stateName);
}

// Issue: State not transitioning
// Check: Are transition conditions met?
private void DebugTransitionConditions()
{
    Debug.Log($"IsGrounded: {_animator.GetBool("IsGrounded")}");
    Debug.Log($"Speed: {_animator.GetFloat("Speed")}");
    Debug.Log($"IsInTransition: {_animator.IsInTransition(0)}");
}
```

### Animator Parameter Issues

```csharp
// Issue: Parameter not updating
// Check 1: Is parameter name spelled correctly? (case-sensitive)
// Check 2: Is parameter type correct?

// Debug: List all parameters
private void LogAnimatorParameters()
{
    foreach (var param in _animator.parameters)
    {
        string value = param.type switch
        {
            AnimatorControllerParameterType.Bool => _animator.GetBool(param.name).ToString(),
            AnimatorControllerParameterType.Float => _animator.GetFloat(param.name).ToString(),
            AnimatorControllerParameterType.Int => _animator.GetInteger(param.name).ToString(),
            AnimatorControllerParameterType.Trigger => "(trigger)",
            _ => "unknown"
        };
        Debug.Log($"Param: {param.name} ({param.type}) = {value}");
    }
}

// Best practice: Cache parameter hashes
private static readonly int SpeedHash = Animator.StringToHash("Speed");
private static readonly int JumpHash = Animator.StringToHash("Jump");

private void SetSpeed(float speed)
{
    _animator.SetFloat(SpeedHash, speed);
}
```

### Animation Event Issues

```csharp
// Issue: Animation event not firing
// Check 1: Is the method public?
// Check 2: Does the method signature match?

// Check 3: Does the name match the event on the clip exactly? (Receivers are named Handle*)

// Valid animation event methods:
public void HandleFootstep() { }                    // No parameters
public void HandleFootstep(string sound) { }        // String parameter
public void HandleFootstep(float volume) { }        // Float parameter
public void HandleFootstep(int index) { }           // Int parameter
public void HandleFootstep(AnimationEvent evt) { }  // Full event data

// Debug: Add logging to event method
public void HandleFootstep(AnimationEvent evt)
{
    Debug.Log($"Footstep event at time {evt.time}, clip: {evt.animatorClipInfo.clip.name}");
}
```

### Root Motion Issues

| Symptom | Check | Solution |
|---------|-------|----------|
| Character not moving | Apply Root Motion enabled? | Enable on Animator component |
| Movement jittery | Mixing root motion with script movement | Use one or the other, not both |
| Wrong movement direction | Animation import settings | Check Bake Into Pose options |
| Sliding feet | Animation doesn't match speed | Adjust animation or movement speed |

```csharp
// Override root motion in script
private void OnAnimatorMove()
{
    // Custom root motion handling
    Vector3 position = _animator.rootPosition;
    Quaternion rotation = _animator.rootRotation;

    // Apply with modifications
    transform.position = position;
    transform.rotation = rotation;
}
```

---

## uGUI Debugging

This project's UI is uGUI (see [UnityUGUIInstructions.md](UnityUGUIInstructions.md), whose
[Troubleshooting](UnityUGUIInstructions.md#troubleshooting) section has the full symptom list).

### Clicks Not Arriving

Work down in order:
1. Is there an `EventSystem` in the scene, with the Input System UI Input Module (not the legacy Standalone one)?
2. Does the Canvas have a `GraphicRaycaster`? World Space canvases also need an **Event Camera**.
3. Is **Raycast Target** on for the element, and `interactable` on for the Selectable?
4. Is a parent `CanvasGroup` hidden with `interactable = false` or `blocksRaycasts = false`?
5. Is something invisible on top of it? A full-screen element with Raycast Target left on is the usual culprit.

- ✅ Select the `EventSystem` during Play mode: its Inspector preview shows the pointer's current event data,
  including which object the pointer is over.
- ✅ For a scripted check, log `EventSystem.RaycastAll` results front to back — the diagnostic is in the uGUI guide's
  Troubleshooting section.

### Rebuilds and Batching

- ✅ Profiler **UI** and **UI Details** modules show which Canvas rebuilt, how often, and its batch count.
- ✅ `Canvas.SendWillRenderCanvases` every frame with nothing visibly changing means something dirties the Canvas
  each frame — an `Animator` on UI, or `.text` / `SetText` called unconditionally.
- ✅ The **Frame Debugger** shows why a batch broke: a different material or texture, a `Mask`, or a sprite-less
  `Image` drawing with `UnityWhite`.

---

## UI Toolkit Debugging (Unity 6+)

Secondary on this project — only relevant if a project uses UI Toolkit.

### Common UI Toolkit Issues

```csharp
// Issue: Element not found
var button = root.Q<Button>("myButton");  // Returns null if not found

// Debug: List all elements
root.Query().ForEach(e => Debug.Log($"{e.GetType().Name}: {e.name}"));

// Issue: Styles not applying
// Check: Is the USS file assigned to the UXML?
// Check: Is the selector correct? (use UI Toolkit Debugger)

// Issue: Events not firing
// Check: Is picking mode set correctly?
button.pickingMode = PickingMode.Position;  // Required for interaction
```

### UI Toolkit Debugger

Access via Window → UI Toolkit → Debugger:
- Inspect live element hierarchy
- View computed styles
- Check class lists and inline styles
- Verify picking mode

### Data Binding Issues (Unity 6)

```csharp
// Verify binding path - it follows the data source's member name
public class PlayerStatsModel
{
    [CreateProperty] public int Health { get; set; }   // Binding path: "Health"
}

// In UXML: data-source-path="Health"
// Common issue: Path is case-sensitive

// Debug: Verify data source is set
var element = root.Q("health-bar");
Debug.Log($"Data source: {element.dataSource}");
Debug.Log($"Data source type: {element.dataSource?.GetType().Name}");
```

### Runtime Data Source Swapping

```csharp
// Pattern: Design-time mock, runtime real data
private void OnEnable()
{
    var root = GetComponent<UIDocument>().rootVisualElement;
    var pane = root.Q("stats-pane");

    // Swap from the design-time mock to the runtime Model (not the Config asset)
    pane.dataSource = _playerStatsModel;
}
```

For comprehensive UI Toolkit patterns, see [UnityUIToolkitInstructions.md](UnityUIToolkitInstructions.md).

---

## Audio Debugging

### AudioSource Issues

```csharp
// Issue: Sound not playing
// Debug: Check AudioSource state
private void DebugAudioSource(AudioSource source)
{
    Debug.Log($"AudioSource on {source.gameObject.name}:");
    Debug.Log($"  Clip: {source.clip?.name ?? "None"}");
    Debug.Log($"  Volume: {source.volume}");
    Debug.Log($"  Mute: {source.mute}");
    Debug.Log($"  IsPlaying: {source.isPlaying}");
    Debug.Log($"  Enabled: {source.enabled}");
    Debug.Log($"  GameObject Active: {source.gameObject.activeInHierarchy}");
}

// Common issues checklist:
// 1. Is AudioClip assigned?
// 2. Is volume > 0?
// 3. Is AudioListener in scene?
// 4. Is AudioSource not muted?
// 5. Is GameObject active?
```

### Spatial Audio Issues

```csharp
// Issue: 3D sound not working
// Check: Spatial Blend setting (0 = 2D, 1 = 3D)
_audioSource.spatialBlend = 1f;   // Fully 3D

// Check: Distance settings
Debug.Log($"Min Distance: {_audioSource.minDistance}");
Debug.Log($"Max Distance: {_audioSource.maxDistance}");

// Debug: Distance to listener
var listener = FindAnyObjectByType<AudioListener>();
if (listener != null)
{
    float distance = Vector3.Distance(transform.position, listener.transform.position);
    Debug.Log($"Distance to listener: {distance}");
}
```

### AudioMixer Issues

```csharp
// Issue: Mixer parameter not changing
// Check: Is parameter exposed? (right-click in Mixer → Expose)

// Debug: Get current mixer values
float value;
if (_mixer.GetFloat("MasterVolume", out value))
{
    Debug.Log($"MasterVolume: {value} dB");
}
else
{
    Debug.LogError("Parameter 'MasterVolume' not found or not exposed");
}

// Note: Mixer uses decibels (-80 to 0), not linear (0 to 1)
// Convert linear to dB:
float LinearToDecibel(float linear)
{
    return linear > 0 ? 20f * Mathf.Log10(linear) : -80f;
}
```

### Common Audio Issues

| Symptom | Check | Solution |
|---------|-------|----------|
| No sound at all | Is AudioListener in scene? | Add AudioListener to camera |
| Sound plays once only | Is Play() called multiple times? | Check loop setting or call Play() again |
| 3D sound always same volume | Is Spatial Blend set to 3D? | Set Spatial Blend to 1 |
| Sound cuts off | Max Distance too small? | Increase Max Distance |
| Mixer not affecting sound | Is AudioSource output set to mixer? | Assign Output in AudioSource |

---

## Async and Coroutine Debugging

UniTask is this project's async default — patterns and rules are in
[UnityUniTaskInstructions.md](UnityUniTaskInstructions.md); this section is for debugging them.

### Coroutine Issues

Coroutines are the rare exception here (see
[When a coroutine is still the right call](UnityUniTaskInstructions.md#when-a-coroutine-is-still-the-right-call)).

```csharp
// Issue: Coroutine stops unexpectedly
// Cause: GameObject disabled or destroyed - a coroutine stops with it

private static readonly WaitForSeconds TickInterval = new(1f);   // cached, not allocated per loop

// Debug: Track coroutine lifecycle
private IEnumerator TickRoutine()
{
    Debug.Log("Coroutine started");
    try
    {
        while (true)
        {
            yield return TickInterval;
            Debug.Log("Coroutine tick");
        }
    }
    finally
    {
        Debug.Log("Coroutine ended");  // Called even when stopped
    }
}

// Issue: Multiple coroutines running
// Fix: Store and stop reference
private Coroutine _tickRoutine;

public void StartTicking()
{
    if (_tickRoutine != null)
        StopCoroutine(_tickRoutine);
    _tickRoutine = StartCoroutine(TickRoutine());
}
```

### Work Continuing After Destroy

```csharp
// Issue: async void + no cancellation - resumes after Destroy(), and its exceptions have nowhere to go
private async void Start()
{
    await Task.Delay(1000);
    transform.position = Vector3.zero;  // May run on a destroyed object
}

// Fix: UniTaskVoid + a destroy token. After Destroy() the await throws OperationCanceledException
// instead of resuming, so no "this == null" checks are needed
private void Start()
{
    TaskResetPosition(this.GetCancellationTokenOnDestroy()).Forget();
}

private async UniTaskVoid TaskResetPosition(CancellationToken token)
{
    try
    {
        await UniTask.Delay(1000, cancellationToken: token);
        transform.position = Vector3.zero;  // Never reached after Destroy()
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

### UniTask Timing Equivalents

```csharp
await UniTask.Delay(TimeSpan.FromSeconds(1), cancellationToken: token);
await UniTask.NextFrame(token);
await UniTask.WaitForEndOfFrame(token);   // Unity 2023.1+: no MonoBehaviour argument needed
await UniTask.WaitForFixedUpdate(token);
```

- ℹ️ `Awaitable` (`Awaitable.WaitForSecondsAsync`, `NextFrameAsync`, …) is Unity-native and valid, but this project
  uses UniTask. Don't mix the two in one flow.

### Finding Leaked or Stuck Tasks

- ✅ Open **Window → UniTask Tracker**. Toggle **Enable Tracking** (low cost) and **Enable StackTrace** (high cost)
  to list every UniTask still running and where it started — a task that should have finished shows up here.
- ⚠️ Debugging only: turn both toggles off when done.

### Exceptions That Vanish

- ⚠️ A `UniTaskVoid` call without `.Forget()` — the compiler warns, and its exceptions go unobserved.
- ⚠️ An exception nobody catches ends up at `UniTaskScheduler.UnobservedTaskException`, which logs it by default
  (`UnobservedExceptionWriteLogType` sets the level). That can be frames after, and far from, the cause.
- ⚠️ `OperationCanceledException` is **silently ignored** there. A cancelled task that was expected to finish
  looks like "nothing happened" — check whether its token was cancelled (object destroyed, scene unloaded).
- ✅ The project rule prevents most of this: every async method has a try/catch that rethrows
  `OperationCanceledException` and logs the rest.

### Coroutine vs UniTask

| Feature | Coroutine | UniTask (project default) |
|---------|-----------|---------------------------|
| Cancellation | `StopCoroutine`; stops when the GameObject is disabled or destroyed | `CancellationToken` (`GetCancellationTokenOnDestroy()`) |
| Return values | Not supported | `UniTask<T>` |
| Exception handling | Limited | try/catch; unhandled → `UnobservedTaskException` |
| Leak inspection | — | UniTask Tracker window |
| Syntax | `yield return` | `async`/`await` |

---

## Event System Debugging

Project events are R3 `Subject`s exposed as `Observable<T>` (see [Events](../UnityStyleGuide.md#events)). Plain C#
events cover Unity's and third-party events and the `event Action` fallback; `UnityEvent` is only for
Inspector-wired callbacks.

### R3 Subscription Issues

```csharp
// Issue: Handler fires once more every time the component is re-enabled
private void OnEnable()
{
    _health.OnHealthChanged.Subscribe(HandleHealthChanged).AddTo(this);  // BAD: not undone on disable
}

// Fix: Subscribe once in Start (or clear an enabled-only CompositeDisposable in OnDisable)
private void Start()
{
    _health.OnHealthChanged.Subscribe(HandleHealthChanged).AddTo(this);
}

// Issue: Handler keeps running after the subscriber is gone
_health.OnHealthChanged.Subscribe(HandleHealthChanged);  // BAD: no AddTo - lives as long as the Subject

// Debug: Log each value on its way to one subscriber
_health.OnHealthChanged
    .Do(onNext: value => Debug.Log($"OnHealthChanged -> {value}", this))
    .Subscribe(HandleHealthChanged)
    .AddTo(this);
```

- ✅ **Window → Observable Tracker** lists every active subscription. Turn on tracking (and stack traces, to see
  where each was made) from the window, or in code with `ObservableTracker.EnableTracking = true` and
  `ObservableTracker.EnableStackTrace = true`. A subscription that should have ended is still listed. Debugging
  only — both default to off.
- ⚠️ `ObjectDisposedException` from a Subject: the owner disposed it in `OnDestroy`, and something raised it or
  subscribed afterwards. Only the owner should call `OnNext`, and only while it is alive.
- ℹ️ An exception thrown inside a handler goes to R3's unhandled-exception handler, which logs it with
  `Debug.LogException` by default. `ObservableSystem.RegisterUnhandledExceptionHandler` replaces it.

### UnityEvent Issues

```csharp
// Issue: UnityEvent not firing
// Debug: Check listener count
Debug.Log($"Listener count: {_onPlayerDeath.GetPersistentEventCount()}");

// Check: Are listeners assigned in Inspector?
// Check: Is the target object not destroyed?
// Check: Is the method signature correct?

// Debug: Log when event fires
[SerializeField] private UnityEvent _onPlayerDeath;

public void Die()
{
    Debug.Log($"Invoking OnPlayerDeath with {_onPlayerDeath.GetPersistentEventCount()} listeners");
    _onPlayerDeath?.Invoke();
}
```

### Plain C# Event Subscription Leaks

Unity's and third-party C# events, and the `event Action` fallback.

```csharp
// Issue: Event keeps firing after object should be done
// Cause: Forgot to unsubscribe

// BAD: Memory leak and potential errors
private void Start()
{
    GameManager.GameOver += HandleGameOver;  // Subscribes
    // Never unsubscribes!
}

// GOOD: Always unsubscribe
private void OnEnable()
{
    GameManager.GameOver += HandleGameOver;
}

private void OnDisable()
{
    GameManager.GameOver -= HandleGameOver;
}

// Debug: Track subscriptions
public static class GameManager
{
    private static event Action _onGameOver;

    public static event Action GameOver
    {
        add
        {
            Debug.Log($"Subscriber added: {value.Target?.GetType().Name}.{value.Method.Name}");
            _onGameOver += value;
        }
        remove
        {
            Debug.Log($"Subscriber removed: {value.Target?.GetType().Name}.{value.Method.Name}");
            _onGameOver -= value;
        }
    }
}
```

### Event Invocation Order

```csharp
// Issue: Events firing in wrong order
// Debug: Log invocation order (plain C# event; an R3 Subject calls subscribers in the order they
// subscribed - log per subscription with .Do(...) as above)

public event Action<int> ScoreChanged;

private void AddScore(int points)
{
    _score += points;

    // Log each subscriber as it's called
    if (ScoreChanged != null)
    {
        int index = 0;
        foreach (var handler in ScoreChanged.GetInvocationList())
        {
            Debug.Log($"Calling handler {index++}: {handler.Target?.GetType().Name}.{handler.Method.Name}");
            ((Action<int>)handler).Invoke(_score);
        }
    }
}
```

### Common Event Issues

| Symptom | Check | Solution |
|---------|-------|----------|
| R3: fires once more per re-enable | `.AddTo(this)` inside `OnEnable`? | Subscribe in `Start`, or clear a `CompositeDisposable` in `OnDisable` |
| R3: handler runs after its object is gone | Subscription without `.AddTo(...)`? | Attach every subscription to a lifetime |
| R3: `ObjectDisposedException` | Subject raised or subscribed after its owner's `OnDestroy`? | Only the owner raises it, while alive |
| C# event never fires | Is Invoke() called? | Add null check and Invoke |
| C# event fires multiple times | Subscribed multiple times? | Unsubscribe in OnDisable |
| NullReference on Invoke | No subscribers? | Use `?.Invoke()` pattern |
| Wrong object receives event | Static event with instance handler? | Unsubscribe on disable/destroy |

---

## ScriptableObject Runtime Issues

When and how to use ScriptableObjects is covered in [UnityScriptableObjectInstructions.md](UnityScriptableObjectInstructions.md); this section is for debugging them.

### Instance vs Asset Confusion

A Config asset is read-only authored data; per-entity runtime state belongs in a plain C# **Model** (see
[MVC layering](UnityArchitectureInstructions.md#mvc-layering)). Most ScriptableObject bugs are runtime state
written into a Config.

```csharp
// Issue: Runtime changes persist in the Editor after Play mode
// Root cause: runtime state written into a Config asset (which exposed a setter)
[SerializeField] private PlayerConfig _playerConfig;

private void TakeDamage(int damage)
{
    _playerConfig.Health -= damage;   // Writes to the ASSET - the change is saved in the Editor
}

// Fix: the Config stays get-only; runtime state lives in a Model created from it
public class PlayerModel
{
    public int Health { get; private set; }

    public PlayerModel(PlayerConfig config) => Health = config.MaxHealth;

    public void ApplyDamage(int amount) => Health = Mathf.Max(0, Health - amount);
}

private PlayerModel _player;

private void Awake()
{
    _player = new PlayerModel(_playerConfig);
}

private void TakeDamage(int damage)
{
    _player.ApplyDamage(damage);   // The asset is never touched
}
```

- ℹ️ While debugging, `Instantiate(_playerConfig)` gives a throwaway copy that stops the asset being edited. Use it
  only to confirm the diagnosis — it isn't the fix, and the copy must be `Destroy`ed.

### Shared Reference Issues

Every object that references a Config sees the same values — that is the point of a Config. If one object's
change shows up in another, something is writing to the shared asset, which is the bug above.

- ✅ To find every writer: make the Config's properties get-only (no setter). The compile errors list each place
  that writes to it; move that state into a Model.

### Common ScriptableObject Issues

| Symptom | Check | Solution |
|---------|-------|----------|
| Changes persist after play mode | Writing to a Config asset? | Move the state into a Model; keep the Config get-only |
| Objects change each other's values | Shared Config being written to? | Same fix — a Config is shared read-only data |
| SO reference null at runtime | Asset not in build? | Check the serialized reference or its Addressables entry |
| OnEnable called in Editor | Normal behavior | Use `Application.isPlaying` check |

---

## Transform and Hierarchy Issues

### Local vs World Space

```csharp
// Issue: Position not where expected
// Debug: Log both spaces
Debug.Log($"Local Position: {transform.localPosition}");
Debug.Log($"World Position: {transform.position}");
Debug.Log($"Parent: {transform.parent?.name ?? "None"}");

// Common confusion:
transform.position = new Vector3(0, 0, 0);      // World space
transform.localPosition = new Vector3(0, 0, 0); // Relative to parent

// Issue: Rotation behaving unexpectedly
Debug.Log($"Local Rotation: {transform.localEulerAngles}");
Debug.Log($"World Rotation: {transform.eulerAngles}");
```

### Parent-Child Relationship Bugs

```csharp
// Issue: Child transform not updating
// Check: Is parent transform being modified?

// Debug: Log hierarchy
private void LogHierarchy(Transform t, int depth = 0)
{
    string indent = new string(' ', depth * 2);
    Debug.Log($"{indent}{t.name} (pos: {t.localPosition}, scale: {t.localScale})");

    foreach (Transform child in t)
    {
        LogHierarchy(child, depth + 1);
    }
}

// Issue: SetParent not working as expected
transform.SetParent(newParent);              // Maintains world position
transform.SetParent(newParent, false);       // Maintains local position
transform.SetParent(newParent, true);        // Maintains world position (explicit)
```

### Scale Inheritance Issues

```csharp
// Issue: Child scaled unexpectedly
// Debug: Check lossy scale (world scale)
Debug.Log($"Local Scale: {transform.localScale}");
Debug.Log($"Lossy Scale: {transform.lossyScale}");  // Accumulated scale

// Issue: Non-uniform parent scale causes skewing
// Fix: Avoid non-uniform scale on parents, or unparent before scaling

// Get world scale without parent influence
Vector3 GetWorldScaleIndependent()
{
    Vector3 scale = transform.localScale;
    Transform parent = transform.parent;

    while (parent != null)
    {
        scale = Vector3.Scale(scale, parent.localScale);
        parent = parent.parent;
    }

    return scale;
}
```

### Common Transform Issues

| Symptom | Check | Solution |
|---------|-------|----------|
| Object in wrong position | Local vs world space? | Use correct position property |
| Object doesn't move with parent | Is it actually a child? | Verify hierarchy in Inspector |
| Scale looks wrong | Parent has non-uniform scale? | Avoid or compensate for parent scale |
| Rotation gimbal lock | Using Euler angles? | Use Quaternion for complex rotations |

---

## Performance Debugging

### Profiler Markers

```csharp
using Unity.Profiling;

private static readonly ProfilerMarker UpdateMarker = new ProfilerMarker("MyScript.Update");

private void Update()
{
    using (UpdateMarker.Auto())
    {
        // Code to profile
    }
}
```

### Common Performance Issues

| Symptom | Diagnostic | Solution |
|---------|------------|----------|
| Frame rate drops | Profiler → CPU | Identify expensive methods |
| Memory grows | Profiler → Memory | Check for leaks, pooling |
| GC spikes | Profiler → GC Alloc | Reduce allocations in Update |
| GPU bound | Frame Debugger | Reduce draw calls, overdraw |

### Allocation-Free Patterns

```csharp
// BAD: Allocates every frame and scans the whole scene
private void Update()
{
    var enemies = FindObjectsByType<Enemy>(FindObjectsSortMode.None); // Scan + array alloc
    string log = $"Count: {enemies.Length}";                          // Allocates string
}

// GOOD: Keep a registry, reuse the builder
private readonly List<Enemy> _activeEnemies = new(100);   // Enemies add/remove themselves
private readonly StringBuilder _statusBuilder = new(64);

private void Update()
{
    _statusBuilder.Clear();
    _statusBuilder.Append("Count: ").Append(_activeEnemies.Count);    // No allocation
}
```

---

## Scene and Asset Debugging

### Missing Reference Detection

```csharp
#if UNITY_EDITOR
[ContextMenu("Find Missing References")]
private void FindMissingReferences()
{
    var fields = GetType().GetFields(
        System.Reflection.BindingFlags.Public |
        System.Reflection.BindingFlags.NonPublic |
        System.Reflection.BindingFlags.Instance);

    foreach (var field in fields)
    {
        if (typeof(UnityEngine.Object).IsAssignableFrom(field.FieldType))
        {
            var value = field.GetValue(this) as UnityEngine.Object;
            if (value == null)
            {
                Debug.LogWarning($"Missing reference: {field.Name}", this);
            }
        }
    }
}
#endif
```

### Prefab Issues

- **Prefab Mode**: Changes in Prefab Mode don't affect scene instances with overrides
- **Nested Prefabs**: Inner prefab changes may not propagate if outer prefab has overrides
- **Prefab Variants**: Check the base prefab for missing references

---

## Build and Platform-Specific Debugging

### Conditional Compilation

```csharp
#if UNITY_EDITOR
    Debug.Log("Editor only");
#endif

#if DEVELOPMENT_BUILD
    Debug.Log("Development build");
#endif

#if UNITY_STANDALONE_WIN
    // Windows-specific code
#elif UNITY_STANDALONE_OSX
    // macOS-specific code
#elif UNITY_IOS
    // iOS-specific code
#elif UNITY_ANDROID
    // Android-specific code
#endif
```

- ℹ️ For logging, don't wrap calls in `#if` — use `AppLogger`, toggled by `ENABLE_LOGS` (see
  [Conditional Logging](#diagnostic-code-patterns)).

### Build-Only Issues

Common causes for "works in Editor, fails in build":
1. **Script stripping**: Add `[Preserve]` attribute or link.xml
2. **Assembly definitions**: Missing references in .asmdef
3. **Resources path**: Case sensitivity on some platforms
4. **Editor-only code**: Wrapped in `#if UNITY_EDITOR`

---

## Collaborative Debugging with AI Tools

AI coding assistants (like Claude Code) cannot directly control IDE debuggers but can effectively assist with debugging through a collaborative workflow.

### What AI Tools Can Do

| Capability | Method |
|------------|--------|
| Read IDE diagnostics | Access compiler errors and warnings via MCP |
| Analyze code flow | Trace logic paths, identify potential issues |
| Add diagnostic code | Insert targeted logging, assertions |
| Interpret errors | Explain stack traces, suggest causes |
| Suggest fixes | Propose code changes based on analysis |

### What Requires Human Interaction

| Capability | Reason |
|------------|--------|
| Set breakpoints | Requires IDE UI interaction |
| Step through code | Real-time interactive process |
| Inspect runtime variables | Requires active debug session |
| Evaluate watch expressions | Requires debugger context |
| View call stack at breakpoint | Requires paused execution |

### Effective Collaboration Workflow

**Step 1: Describe the Issue**
```
Error: NullReferenceException in PlayerMover.Update() line 47
Repro: Start game, press jump button
Expected: Player jumps
Actual: Exception thrown, player frozen
```

**Step 2: AI Analyzes Code**
The AI reads relevant files, traces execution flow, and identifies suspects.

**Step 3: AI Adds Diagnostic Code**
```csharp
private void Update()
{
    // Diagnostic logging added by AI
    Debug.Log($"[DEBUG] _inputHandler: {_inputHandler != null}", this);
    Debug.Log($"[DEBUG] _characterController: {_characterController != null}", this);
    Debug.Log($"[DEBUG] _isGrounded: {_isGrounded}", this);

    HandleMovement();
    HandleJump();  // Line 47 - exception occurs here
}
```

**Step 4: Human Runs and Reports**
Share the console output with the AI.

**Step 5: AI Interprets and Fixes**
Based on logs, the AI identifies the null reference and proposes a fix.

### Programmatic Breakpoints

When you need the debugger to pause at a specific condition:

```csharp
private void Update()
{
    // Pause debugger when unexpected state occurs
    if (_health < 0)
    {
        Debug.LogError("Health went negative - breaking to debugger");
        System.Diagnostics.Debugger.Break();  // Rider/VS will pause here
    }

    // Conditional break with context logging
    if (_player == null && _wasPlayerValid)
    {
        Debug.LogError($"Player reference lost! Last valid frame: {_lastValidFrame}");
        System.Diagnostics.Debugger.Break();
    }
}
```

### Diagnostic Code Patterns

**Trace Method Entry/Exit**
```csharp
private void ProcessInput()
{
    Debug.Log($"→ {nameof(ProcessInput)} entered", this);

    // ... method logic ...

    Debug.Log($"← {nameof(ProcessInput)} exiting", this);
}
```

**State Snapshot**
```csharp
[ContextMenu("Log State Snapshot")]
private void LogStateSnapshot()
{
    Debug.Log("=== STATE SNAPSHOT ===");
    Debug.Log($"Position: {transform.position}");
    Debug.Log($"Velocity: {_rigidbody?.linearVelocity}");
    Debug.Log($"IsGrounded: {_isGrounded}");
    Debug.Log($"CurrentState: {_currentState}");
    Debug.Log($"InputVector: {_inputVector}");
    Debug.Log("======================");
}
```

**Conditional Logging (`ENABLE_LOGS`)**

Diagnostic logging you keep goes through `AppLogger`. Its `Log` and `LogWarning` are
`[Conditional("ENABLE_LOGS")]`, so the call and the string built for it are removed from any build without the
define — including development and QA builds, which is why it's `ENABLE_LOGS` and not `UNITY_EDITOR`.

```csharp
private const string DebugPrefix = "[Inventory]";

// Stripped (call and string) when ENABLE_LOGS isn't defined
AppLogger.Log($"{DebugPrefix} Processing {_items.Count} items", this);

// A class-local helper needs the same attribute, or the string is still built at the call site
[System.Diagnostics.Conditional("ENABLE_LOGS")]
private void DebugLog(string message) => AppLogger.Log($"{DebugPrefix} {message}", this);
```

### Tips for Effective AI-Assisted Debugging

1. **Provide context**: Include error messages, stack traces, and repro steps
2. **Share console output**: Copy/paste the actual log output after running
3. **Describe expected vs actual**: What should happen vs what does happen
4. **Mention recent changes**: What was modified before the bug appeared
5. **Include relevant settings**: Unity version, platform, Enter Play Mode Options

---

## Learn more

- [Debugging C# code in Unity](https://docs.unity3d.com/6000.3/Documentation/Manual/managed-code-debugging.html)
- [Console window](https://docs.unity3d.com/6000.3/Documentation/Manual/Console.html)
- [Frame Debugger](https://docs.unity3d.com/6000.3/Documentation/Manual/FrameDebugger.html)
- [UniTask Tracker](https://github.com/Cysharp/UniTask#unitasktracker)
- [R3 ObservableTracker](https://github.com/Cysharp/R3#observabletracker)
