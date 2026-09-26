# AGENTS.md — Unity 6 C#

Coding conventions for this Unity project. Copy this file into your own repo and edit the
**Project Setup** block below — everything else is ready to use as-is.

Rules marked `[opinion]` are stylistic preferences, not industry consensus. Change them freely as
long as you change them everywhere. Everything unmarked is standard Unity or C# practice and should
survive contact with any team.

---

## Project Setup — EDIT THIS BLOCK

```
Unity version:    6.3 (6000.3.x)
C# version:       9.0
Render pipeline:  URP
Input:            Input System package  (NOT the legacy Input Manager)
UI:               uGUI                  (NOT UI Toolkit)
Async:            UniTask               (NOT Awaitable/coroutines, except where per-frame iteration is genuinely needed)
Pooling:          UnityEngine.Pool.ObjectPool<T>
```

Never generate code for a system listed as "NOT". If the project uses something different, change
the block above first.

---

## Never do these — they corrupt the project

Style mistakes produce ugly code. These produce a broken Unity project, and the damage is often
silent until someone opens the Editor.

- ❌ **Never create or hand-edit a `.meta` file.** Unity owns the GUID inside it. A wrong or
  duplicated GUID breaks every reference to that asset, across every scene and prefab.
- ❌ **Never edit `.asset`, `.prefab`, `.unity`, `.controller`, `.mat`, or `.anim` YAML directly.**
  They carry `fileID` + GUID pairs that must stay internally consistent. Use the Editor, or an
  Editor script that goes through `AssetDatabase` / `SerializedObject`.
- ❌ **Never move, rename, or delete an asset with filesystem commands** (`mv`, `rm`, `git mv`). The
  `.meta` must travel with it. Use the Project window or `AssetDatabase.MoveAsset`.
- ❌ **Never rename a serialized field without `[FormerlySerializedAs("oldName")]`.** Serialization
  is keyed on the field name — the Inspector value is silently lost on every asset and scene using it.
- ❌ **Never reorder or renumber the members of a serialized enum.** They persist as integers, so
  existing assets silently remap to different values. Append new members at the end.
- ⚠️ Don't edit `ProjectSettings/*.asset` or `Packages/manifest.json` while the Editor is open — it
  may overwrite your change when it regains focus.
- ⚠️ `Library/`, `Temp/`, `obj/`, `Logs/` and `UserSettings/` are generated. Never edit them, never
  commit them.

✅ When serialized data genuinely has to change in bulk, write an Editor script. It goes through the
same code path the Inspector does, so GUIDs, fileIDs, prefab overrides and Undo all stay correct.
See `UnityReferenceGuides/UnityEditorToolingInstructions.md`.

## Deprecated in Unity 6

Never write the left column. These are renamed or obsolete, and the old names dominate tutorials
and training data — they are the single most common source of stale generated code.

| Don't write | Write |
|---|---|
| `FindObjectOfType` | `FindAnyObjectByType` |
| `FindObjectsOfType` | `FindObjectsByType<T>()` † |
| `Rigidbody.velocity` | `Rigidbody.linearVelocity` |
| `Rigidbody.drag` / `.angularDrag` | `linearDamping` / `angularDamping` |
| `Rigidbody2D.velocity` / `.drag` / `.angularDrag` | `linearVelocity` / `linearDamping` / `angularDamping` |
| `Graphics.DrawMesh` | `Graphics.RenderMesh(RenderParams, …)` |
| `Graphics.DrawMeshInstanced` | `Graphics.RenderMeshInstanced` |
| `Graphics.DrawMeshInstancedIndirect` | `Graphics.RenderMeshIndirect` |
| `ParticleSystem.emissionRate` | `emission.rateOverTime` |
| `ParticleSystem.enableEmission` | `emission.enabled` |
| `TerrainData.splatPrototypes` | `terrainLayers` |
| `UxmlFactory` / `UxmlTraits` | `[UxmlElement]` / `[UxmlAttribute]` |
| `VisualElement.ExecuteDefaultAction` | `HandleEventBubbleUp` / `HandleEventTrickleDown` |
| `EventBase.PreventDefault()` | `StopPropagation()` |
| `GraphicsFormat.DepthAuto` / `ShadowAuto` / `VideoAuto` | `GraphicsFormat.None` + `RenderTextureFormat.Depth` / `Shadowmap` |
| `ScriptableRenderPass.Execute` | `RecordRenderGraph(RenderGraph, ContextContainer)` |
| `RenderTargetHandle` | `RTHandle` / `TextureHandle` |
| `ScriptableRenderer.cameraColorTarget` | `cameraColorTargetHandle` |
| `[CustomEditorForRenderPipeline]` | `[CustomEditor]` + `[SupportedOnRenderPipeline]` |
| `[VolumeComponentMenuForRenderPipeline]` | `[VolumeComponentMenu]` + `[SupportedOnRenderPipeline]` |
| `LightingSettings.filteringGaussRadius*` | `filteringGaussianRadius*` (now `float`) |

⚠️ `ParticleSystem` modules are structs that write through to native memory. `ps.emission.enabled =
true` does not compile — copy to a local, mutate the local, and the change applies:
`var emission = ps.emission; emission.enabled = true;`

† `FindFirstObjectByType` and every `FindObjectsByType` overload taking a `FindObjectsSortMode`
(including `FindObjectsByType<T>(FindObjectsSortMode.None)`) became Obsolete in **6000.6** — they
were the correct replacements for `FindObjectOfType`/`FindObjectsOfType` from Unity 2023.1 through
6000.5, so don't flag them on an older project. The parameterless `FindObjectsByType<T>()` (and
`FindObjectsByType<T>(FindObjectsInactive)`) did not exist before 6000.6 — check
`UnityCustomInstructions/UnityTechStack.md` for the project's actual version before applying this
row, and keep the `FindObjectsSortMode.None` form on 6000.0–6000.5.

## If you read nothing else

1. `_camelCase` for private fields and mutable statics; PascalCase for constants and `static readonly` values. `[opinion]`
2. PascalCase for types, methods, properties, enums, tags, layers, and animator parameters.
3. Booleans read as predicates: `isDead`, `hasKey`, `CanJump`.
4. Fields are private. Expose with `[SerializeField]` for the Inspector, properties for other classes.
5. Allman braces. Always write `private` explicitly. `[opinion]`
6. Cache in `Awake`. Every subscription has a guaranteed end: R3 `.AddTo(this)`, or `+=` in `OnEnable` paired with **`-=` in `OnDisable` — always.**
7. Nothing in `Update` may allocate, call `GetComponent`/`Find*`/`Camera.main`, or run LINQ.
8. No magic numbers or strings. Use constants or serialized fields.
9. Comment *why*, not *what*. Prefer a clearer name to a comment.
10. Prefer composition over inheritance; keep MonoBehaviours single-purpose.

---

## Naming

| Element | Convention | Example |
|---|---|---|
| Private field | `_` + camelCase `[opinion]` | `_currentHealth` |
| Mutable static field | `_` + camelCase `[opinion]` | `_instance` |
| Constant (`const`, any accessibility) | PascalCase | `MaxRetries`, `Tags.Player` |
| `static readonly` value | PascalCase | `SpeedHash`, `UpdateMarker` |
| Property | PascalCase | `CurrentHealth` |
| Method | PascalCase verb | `ApplyDamage` |
| Observable (R3) | `On` + past tense `[opinion]` | `OnHealthChanged` |
| Event (`event Action` fallback) | Past tense, no `On` | `HealthChanged` |
| Interface | `I` + PascalCase | `IDamageable` |
| Enum type / values | PascalCase, singular noun | `Direction.North` |
| Class / file / folder | PascalCase | `PlayerController.cs` |
| ScriptableObject | PascalCase + `Config` `[opinion]` | `WeaponConfig` |
| Async method | `Task`-prefix, PascalCase `[opinion]` | `TaskLoadLevel` |
| Coroutine | PascalCase + `Routine` `[opinion]` | `FadeOutRoutine` |
| Animator param / tag / layer | PascalCase | `IsRunning` |

- Include units when a number is ambiguous: `_speedInMetersPerSecond`, `_delayInSeconds`.
- Don't repeat the class name in a member: in `Player`, use `_health` not `_playerHealth`.
- No abbreviations unless universally known (`UI`, `ID`).
- Boolean methods pose a question: `IsPlayerAlive()`, `HasLineOfSight()`.
- Method verbs: `Set*` assigns, `Change*` transforms, `Handle*` responds to an event,
  `Process*` runs game-flow logic. `[opinion]`
- Avoid gerunds — `Walking()` describes state, not an action. Use `Walk()`, or `IsWalking` as a property.

## Formatting

- Allman braces (opening brace on its own line). `[opinion]`
- 4-space indent. Max line width 120–140. `[opinion]`
- One variable declaration per line.
- Space after commas and around operators; no space inside brackets or before an argument list.
- `using` order: `System`, then Unity, then project namespaces. Remove unused ones.
- Blank lines between logical blocks.
- `#region` only for animation/input event handler groups. Never to hide length.

## Class layout

Members appear in Unity's execution order:

```
using directives
namespace
  fields
  properties
  events
  Awake, OnEnable, Start, OnDisable, OnDestroy, FixedUpdate, Update, LateUpdate
  public methods
  private methods (Handle* callbacks grouped together)
  small helper types used only by this file
```

One MonoBehaviour per file, named after the file.

## MonoBehaviour lifecycle

| Method | Do | Don't |
|---|---|---|
| `Awake` | Cache own components, init internal state | Touch other objects, subscribe |
| `OnEnable` | Subscribe to plain C# events (`+=`), or R3 subscriptions that must stop while disabled; reset per-enable state | Heavy work; `.AddTo(this)` (not undone on disable, so re-enabling duplicates it) |
| `Start` | Anything needing other objects to exist; R3 subscriptions (`.AddTo(this)`) | Work that belongs in `Awake` |
| `OnDisable` | Unsubscribe — mirror `OnEnable` exactly (`-=`, or `Clear()` the enabled-only subscriptions) | Release native resources |
| `OnDestroy` | Dispose native resources, `IDisposable` fields, and owned `Subject`s | Unsubscribe plain C# events (too late; use `OnDisable`) |
| `FixedUpdate` | Physics, forces, velocity | Read input; allocate |
| `Update` | Input, timers, state machines | Allocate, `GetComponent`, `Find*`, LINQ, `Camera.main` |
| `LateUpdate` | Camera follow, post-Update corrections | Anything another script must read this frame |

- Use `[RequireComponent(typeof(T))]` for hard dependencies instead of runtime null checks.
- Order within a lifecycle phase is undefined. Don't rely on it.

## Fields, properties and serialization

```csharp
[SerializeField] private int _maxHealth = 100;       // Inspector-visible, still private
public int MaxHealth => _maxHealth;                  // Read-only access for others

[field: SerializeField] public float MoveSpeed { get; private set; } = 5f;  // Concise alternative
```

- Never `public` fields. Never serialize a property directly.
- Properties for lightweight state access; methods for actions and anything with side effects.
  `ApplyDamage(10)`, not `Health = Health - 10`.
- No redundant initializers — value types are already `0`, references already `null`.

## Events

R3 is the default for events and notifications (see [Architecture](#architecture)). A class owns a private
`Subject<T>` and exposes it as a read-only `Observable<T>`; each subscriber attaches its subscription to its own
lifetime.

```csharp
// Publisher: owns the Subject, exposes only the Observable
private readonly Subject<int> _healthChanged = new();
public Observable<int> OnHealthChanged => _healthChanged;

public void ApplyDamage(int amount)
{
    _health -= amount;
    _healthChanged.OnNext(_health);
}

private void OnDestroy() => _healthChanged.Dispose();

// Subscriber: in Start; the subscription ends when this object is destroyed
private void Start() => _health.OnHealthChanged.Subscribe(HandleHealthChanged).AddTo(this);
```

- Expose `Observable<T>`, never the `Subject<T>` — only the owner raises it. `Subject<Unit>` for an event with no
  payload.
- Observable: `On` + past tense (`OnDoorOpened`). Handler: `Handle*`. `[opinion]`
- Subscribe in `Start` with `.AddTo(this)`. **Never `.AddTo(this)` in `OnEnable`** — it isn't undone on disable, so
  every re-enable adds a duplicate. A handler that must stop while the component is disabled subscribes in
  `OnEnable` into a `CompositeDisposable` that `OnDisable` `Clear()`s.
- A plain C# subscriber (Controller, service) holds a `DisposableBag` field, subscribes with
  `.AddTo(ref _subscriptions)`, and disposes it in its own `Dispose()`. A bag is a struct: keep the field
  non-`readonly` and never copy it. `CompositeDisposable` is for the enabled-only case above — its `Clear()`
  keeps its backing list, while a bag re-allocates its array after every `Clear()`.
- The owner disposes its `Subject` in `OnDestroy`. Raising or subscribing after that throws
  `ObjectDisposedException`.
- A `readonly struct` for multi-value payloads: `Subject<DamageInfo>`.
- Prefer `Subject`/`Observable` over `UnityEvent` for every code-driven event. `UnityEvent` only for events exposed to
  the Inspector for designers to wire. To consume a Unity component's `UnityEvent` from code, convert it
  (`_button.OnClickAsObservable()`, or `.AsObservable(destroyCancellationToken)`) instead of `AddListener`.
- Use events to decouple unrelated systems. Use a direct call when one object owns the other.
- Across features, the default is a directly injected interface, not an event or a global bus — see
  [Architecture](#architecture).

**Fallback — `event Action`, only where R3 can't be referenced** (e.g. an assembly that must not depend on it):

```csharp
public event Action<int> HealthChanged;

protected virtual void OnHealthChanged(int value) => HealthChanged?.Invoke(value);

private void OnEnable()  => _health.HealthChanged += HandleHealthChanged;
private void OnDisable() => _health.HealthChanged -= HandleHealthChanged;
```

- Past-tense event names (`DoorOpened`); `On`-prefixed raiser methods (`OnDoorOpened`).
- Always `?.Invoke()`. Never subscribe with a lambda — you can't unsubscribe it.
- Unity's and third-party C# events (Input System actions, `SceneManager.sceneLoaded`) stay `+=` in `OnEnable`,
  `-=` in `OnDisable`.

## Performance rules

Non-negotiable in `Update`, `FixedUpdate`, `LateUpdate`, and any per-frame path:

- No `new` on reference types. Pre-allocate and `Clear()`.
- No `GetComponent`, `FindObjectsByType`, `GameObject.Find`, or `Camera.main`. Cache in `Awake`.
- No LINQ. No string concatenation or interpolation.
- No `Physics.OverlapSphere`/`Raycast` without a layer mask, a max distance, and preferably a
  non-allocating overload.
- Cache `Shader.PropertyToID` and `Animator.StringToHash` results in `static readonly int` fields.
- Cache `WaitForSeconds`; never allocate one per loop iteration.
- Use `CompareTag("X")` rather than `tag == "X"`.
- Use `.sharedMaterial`, not `.material`, unless you intend to create an instance.
- Pool anything spawned frequently with `UnityEngine.Pool.ObjectPool<T>`.
- Throttle expensive work with a timer, or spread it across frames.

Unity 6 API notes:

- Renamed and obsolete APIs are listed under [Deprecated in Unity 6](#deprecated-in-unity-6).
  Beyond the rename, prefer a registry over any `Find*` call at runtime.
- Prefer UniTask over `Awaitable`/coroutines for gameplay async code — see [Async](#async) below.
  `Awaitable` is still valid Unity 6 API and works the same way for a project that isn't on UniTask.
- Avoid `async void`; use `UniTaskVoid` (fire-and-forget) or `UniTask`/`UniTask<T>` (awaited) so
  exceptions can actually be observed.

## Async

- UniTask is this project's default for async gameplay code, per
  [`UnityTechStack.md`](UnityCustomInstructions/UnityTechStack.md). Prefix async methods with
  `Task`, never suffix with `Async`: `TaskOpenDoor`, not `OpenDoorAsync`. `[opinion]`
- Pull a `MonoBehaviour`'s cancellation token from `this.GetCancellationTokenOnDestroy()` rather
  than managing one by hand.
- Wrap every async method body in `try`/`catch (System.Exception e)`, not just where there's
  fallback logic — re-throw `OperationCanceledException` explicitly. Stricter than the general
  [Control flow](#control-flow) try/catch rule, which still governs synchronous code.
- The rare legitimate coroutine takes a `Routine` suffix (`FadeOutRoutine`), not `Task`-prefix.
- Full pattern set — cancellation, error handling, `WhenAll`/`WhenAny`, closure-free `WaitUntil`,
  and more — in
  [`UnityReferenceGuides/UnityUniTaskInstructions.md`](UnityReferenceGuides/UnityUniTaskInstructions.md).

```csharp
public async UniTask TaskOpenDoor()
{
    try
    {
        await UniTask.Delay(2000, cancellationToken: this.GetCancellationTokenOnDestroy());
        _isOpen = true;
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

## Architecture

- MVC per feature. **Model** = plain C# `struct`/class holding one entity's data. **View** =
  `MonoBehaviour`, display only. **Controller** = plain C# class holding the logic, never a
  `MonoBehaviour`. Feature-wide settings are a `*Config` ScriptableObject, not the Model.
- Every Controller has an interface. A View has one where feasible. A Model only when the data is
  more than trivial.
- Wire services with VContainer constructor injection — never `new` a service, never a static locator.
  A boot scene that loads first and is never unloaded hosts the `LifetimeScope` that registers the
  app-lifetime singletons; every later scene loads additively (never `Single`) and its scope is parented
  to it with `LifetimeScope.EnqueueParent`.
- Singletons (`Instance` statics) are discouraged. If one is unavoidable, state the reason in a
  comment at the declaration.
- Talk to other systems through a directly injected interface. MessagePipe is optional, and only for
  circular dependencies, fan-out to an unknown number of listeners, mismatched lifetimes, or ownerless
  broadcast events.
- R3 `Subject`/`Observable` is the default for events and notifications, subscribed with `.AddTo(...)`.
  Plain `event Action` is the fallback only where R3 can't be referenced.
- Details:
  [`UnityReferenceGuides/UnityArchitectureInstructions.md`](UnityReferenceGuides/UnityArchitectureInstructions.md).

## ScriptableObjects

- For static, authored configuration and shared content. **Not** for runtime state — Editor edits
  persist between play sessions. Per-entity runtime data belongs in a plain C# class or struct.
- Always `[CreateAssetMenu(fileName = "<ClassName>", menuName = "<Category>/<Asset Name>")]`, e.g.
  `menuName = "Enemies/Enemy Config"`. `fileName` matches the class name exactly.
- Expose data through get-only properties over `[SerializeField]` private fields, never public fields.
- Suffix with `Config`. `[opinion]`
- ℹ️ Details:
  [`UnityReferenceGuides/UnityScriptableObjectInstructions.md`](UnityReferenceGuides/UnityScriptableObjectInstructions.md).

## Strings and collections

- Interpolation over concatenation for **readability** — both allocate. In hot paths, cache the
  result and rebuild only when the value changes; use `StringBuilder` for multi-part assembly.
- `List<T>` for dynamic size, arrays for fixed size in tight loops, `Dictionary<K,V>` for lookups,
  `Stack<T>` for LIFO.
- Give collections an initial capacity when you know it. Reuse and `Clear()` instead of reallocating.

## Control flow

- Guard clauses over nested `if`. `if (!ready) return;`
- Enums for mutually exclusive state, never magic ints or strings.
- `try`/`catch` only for genuinely external failures (file I/O, network). Validate inputs instead of
  catching predictable problems. **Async methods are the exception — every one gets a try/catch,
  see [Async](#async).**
- Never leave `throw new NotImplementedException()` in a generated stub. Empty body or a comment.

## Editor code

- Editor-only classes go in an `Editor/` folder; Editor-only members go behind `#if UNITY_EDITOR`.
- Never `#if UNITY_EDITOR` around a serialized field — it changes serialization between Editor and build.
- In custom inspectors, always go through `SerializedProperty`. Call `Undo.RecordObject` **before**
  the change, `EditorUtility.SetDirty` after.
- Prefer `OnDrawGizmosSelected` over `OnDrawGizmos` — the latter runs every Editor frame for every
  instance.

## Logging

- Tag each message with a class- or module-level `DebugPrefix` constant: `$"{DebugPrefix} message"`.
- Match the call to the severity: `Debug.Log`, `Debug.LogWarning`, `Debug.LogError`.
- Pass the context object as the second argument so a click selects it in the Hierarchy.
- Route project logging through a wrapper (`AppLogger`). Its `Log` and `LogWarning` are marked
  `[System.Diagnostics.Conditional("ENABLE_LOGS")]`. `Debug.Log` on its own is not stripped from builds,
  and its string is built even when logging is off; `[Conditional]` removes the call and the string.
- `ENABLE_LOGS` is the one define that toggles `Log` and `LogWarning`. Add it to Scripting Define Symbols
  for builds that should log (Editor, development, QA); leave it out of release builds. `LogError` is
  deliberately not `[Conditional]`, so errors survive in release.
- No logging in `Update`-style hot paths.

## Statics and domain reload

If Enter Play Mode Options has Domain Reload disabled, static state survives between play sessions.
Every class with statics needs a reset:

```csharp
[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]
private static void ResetStatics()
{
    _score = 0;
    ScoreChanged = null;   // Static events keep their subscribers otherwise
}
```

## Input, animation, audio

- **Input System:** an action that was never `Enable()`d fails silently. Subscribe in `OnEnable`,
  unsubscribe in `OnDisable`, never with a lambda. Read input in `Update`, apply in `FixedUpdate`.
  Value actions need a `canceled` handler or movement sticks. Dispose the generated input class.
- **Animation:** always `Animator.StringToHash` into a `static readonly int`; a mistyped parameter
  name fails silently. `ResetTrigger` before `SetTrigger` on interruptible actions. One component
  owns a given parameter. Culling Mode off `AlwaysAnimate`.
- **Audio:** every `AudioSource` needs an `outputAudioMixerGroup`. Never `PlayClipAtPoint` in a hot
  path — pool sources. A 0–1 slider must go through `Mathf.Log10(v) * 20` to reach a dB parameter,
  guarded against zero. `spatialBlend = 1` for positional sound. Exactly one `AudioListener`.
- **Assemblies:** dependencies point inward — nothing references UI. Editor assemblies need
  `"includePlatforms": ["Editor"]`; test assemblies need `UNITY_INCLUDE_TESTS`.

## Physics

- Never assign `transform.position` on an active non-kinematic Rigidbody — it teleports the body,
  discards velocity and skips continuous collision detection. Use `AddForce`, or `MovePosition` on
  a kinematic body.
- Forces and velocity go in `FixedUpdate` with an explicit `ForceMode`. Read input in `Update` and
  latch it; `FixedUpdate` may run zero or many times per frame.
- `Time.fixedDeltaTime` inside `FixedUpdate`, never `Time.deltaTime`.
- **A collider pair with no Rigidbody on either side sends no messages.** A static trigger volume
  needs the *moving* object to carry the Rigidbody; give a script-moved detector a kinematic one.
- `OnCollisionEnter(Collision)` and `OnTriggerEnter(Collider)` — the wrong parameter type means
  Unity silently never calls the method.
- Tunnelling needs both sides set: the fast body `ContinuousDynamic`, what it hits `Continuous`.
  Continuous detection only works on Sphere, Capsule and Box colliders.
- `Interpolate` on the body the camera follows, `None` on the rest.
- Convex MeshColliders cap at 255 triangles; non-convex can't be a trigger and can't go on a
  non-kinematic Rigidbody. Prefer primitives and compounds.
- Prune the Layer Collision Matrix — unchecked pairs are skipped before any contact work.

## UI

**uGUI** (this project's default, per `UnityCustomInstructions/UnityTechStack.md`) — split canvases by
update frequency; turn off `Raycast Target` on non-interactive graphics; never nest layout groups;
never put an `Animator` on UI; hide a screen with a `CanvasGroup` (alpha 0, not interactable, no raycasts)
rather than `SetActive` — unless the hidden screen runs heavy per-frame logic that must stop; set numbers
with TMP `SetText("{0}", value)`, not `.text = value.ToString()`; consume `onClick` with
`OnClickAsObservable()`, not `AddListener`; `RectMask2D` over `Mask`; pool list rows.

**UI Toolkit** (secondary — only if the project switches to it) — kebab-case BEM for UXML `name` and
`class` (`block__element--modifier`). Use `AddToClassList` / `EnableInClassList`, never
`classList.Add`. USS is not CSS: no `gap`, no `calc()`, no `z-index`, no hex colours, no `em`/`rem`,
flexbox only. Register callbacks in `OnEnable`, unregister in `OnDisable`.

## Comments

- Explain intent, not mechanics. If a comment restates the code, delete one of them.
- `[Tooltip]` beats a comment on any serialized field a designer will touch.
- XML docs (`///`) on public APIs.
- No `// Created by` — that's what version control is for.

---

## Deeper reference

| Topic | File |
|---|---|
| Full style guide with examples | `UnityStyleGuide.md` |
| Performance, profiling, Jobs, rendering | `UnityReferenceGuides/UnityPerformanceOptimizationInstructions.md` |
| SOLID and design patterns | `UnityReferenceGuides/UnityDesignPatternsInstructions.md` |
| Architecture: MVC, VContainer DI, Singleton policy, MessagePipe, R3 | `UnityReferenceGuides/UnityArchitectureInstructions.md` |
| ScriptableObjects, `[CreateAssetMenu]`, config vs runtime data | `UnityReferenceGuides/UnityScriptableObjectInstructions.md` |
| UniTask patterns, cancellation, closure-free overloads | `UnityReferenceGuides/UnityUniTaskInstructions.md` |
| UI Toolkit (UXML/USS/BEM) | `UnityReferenceGuides/UnityUIToolkitInstructions.md` |
| uGUI / Canvas | `UnityReferenceGuides/UnityUGUIInstructions.md` |
| Assets, Addressables, memory | `UnityReferenceGuides/UnityAssetsAndMemoryInstructions.md` |
| Scenes, bootstrapping, lifecycle | `UnityReferenceGuides/UnityScenesAndLifecycleInstructions.md` |
| Testing | `UnityReferenceGuides/UnityTestingInstructions.md` |
| Editor tooling | `UnityReferenceGuides/UnityEditorToolingInstructions.md` |
| Input System actions, phases, rebinding | `UnityReferenceGuides/UnityInputSystemInstructions.md` |
| Rigidbodies, colliders, collision callbacks, queries | `UnityReferenceGuides/UnityPhysicsInstructions.md` |
| Animator parameters, blend trees, events | `UnityReferenceGuides/UnityAnimationInstructions.md` |
| Mixers, AudioSource pooling, spatial audio | `UnityReferenceGuides/UnityAudioInstructions.md` |
| Assembly definitions and dependency boundaries | `UnityReferenceGuides/UnityAssemblyDefinitionsInstructions.md` |
| Debugging | `UnityReferenceGuides/UnityDebuggingInstructions.md` |
| Project-specific settings | `UnityCustomInstructions/` |
