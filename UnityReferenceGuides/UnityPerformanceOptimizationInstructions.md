# Unity Performance Optimization Instructions for LLM Coding Tools

> **General Unity best practice.** Applies to any Unity 6 project. Change these only if you
> know why. Personal style preferences live in [`UnityStyleGuide.md`](../UnityStyleGuide.md);
> project-specific settings live in [`UnityCustomInstructions/`](../UnityCustomInstructions/).


Use this guide when reviewing Unity projects for performance issues. These instructions help identify common bottlenecks and suggest optimizations that Claude can apply when analyzing code structure and project organization.

Related cost guidance lives with its topic: texture, mesh and audio import settings and Addressables memory in
[UnityAssetsAndMemoryInstructions.md](UnityAssetsAndMemoryInstructions.md); physics settings in
[UnityPhysicsInstructions.md](UnityPhysicsInstructions.md); Animator setup in
[UnityAnimationInstructions.md](UnityAnimationInstructions.md).

Table of contents:
- [Unity Version-Specific Notes](#unity-version-specific-notes)
- [Code Review Priority Checklist](#code-review-priority-checklist)
- [Update Loop Optimization](#update-loop-optimization)
    - [Avoiding Per-Frame Allocations](#avoiding-per-frame-allocations)
    - [Caching Expensive Operations](#caching-expensive-operations)
    - [Throttling Update Logic](#throttling-update-logic)
    - [Update Loop Discipline](#update-loop-discipline)
- [Memory Management](#memory-management)
    - [String Operations](#string-operations)
    - [Collections and Allocations](#collections-and-allocations)
    - [Boxing and Unboxing](#boxing-and-unboxing)
    - [Loops Over Collections](#loops-over-collections)
    - [Unity Objects You Own](#unity-objects-you-own)
- [Object Pooling](#object-pooling)
- [Physics Optimization](#physics-optimization)
- [Rendering Considerations](#rendering-considerations)
- [GetComponent and Find Operations](#getcomponent-and-find-operations)
- [LINQ and Delegates](#linq-and-delegates)
- [Async and Coroutine Patterns](#async-and-coroutine-patterns)
- [Data Structure Selection](#data-structure-selection)
- [Unity API Best Practices](#unity-api-best-practices)
- [Profiling Markers](#profiling-markers)
- [Garbage Collection](#garbage-collection)
- [Jobs and Burst](#jobs-and-burst)
- [Rendering Performance](#rendering-performance)
    - [SRP Batcher](#srp-batcher)
    - [GPU Resident Drawer and occlusion culling](#gpu-resident-drawer-and-occlusion-culling)
    - [Frame pacing](#frame-pacing)
    - [Batching settings](#batching-settings)
    - [Culling](#culling)
    - [Shadows and lighting](#shadows-and-lighting)
    - [LOD and mipmap streaming](#lod-and-mipmap-streaming)
    - [Render resolution](#render-resolution)
    - [Particle systems](#particle-systems)
    - [Animator cost](#animator-cost)
    - [UI](#ui)
- [Profiling Workflow](#profiling-workflow)
    - [Profiling checklist](#profiling-checklist)
- [Common Anti-Patterns](#common-anti-patterns)
- [Troubleshooting](#troubleshooting)
- [Learn more](#learn-more)

---

# Unity Version-Specific Notes

- ℹ️ This project's async default is UniTask — see [UnityUniTaskInstructions.md](UnityUniTaskInstructions.md). Unity 6's
  `Awaitable` is a valid Unity-native alternative for a project that isn't on UniTask.
- ℹ️ Unity 6's `UnityEngine.Pool.ObjectPool<T>` should be preferred over custom pooling implementations.
- ℹ️ The Burst compiler can dramatically improve performance for math-heavy code when used with the Jobs system.
- ℹ️ IL2CPP builds have different performance characteristics than Mono—profile on target platform.

---

# Code Review Priority Checklist

When reviewing Unity code for performance, check these areas in order of impact:

1. **Update loops** — Allocations, expensive operations, unnecessary work
2. **Physics** — OverlapSphere, Raycast frequency, collision matrix
3. **Memory** — String concatenation, LINQ in hot paths, boxing
4. **GetComponent/Find** — Uncached lookups, per-frame calls
5. **Rendering** — Material instances, shader keywords, draw calls

---

# Update Loop Optimization

## Avoiding Per-Frame Allocations

- ❌ **Never allocate in Update(), FixedUpdate(), or LateUpdate()** — This triggers garbage collection spikes.
- ❌ Avoid `new` keyword for reference types in update loops.
- ❌ Avoid string concatenation or interpolation in update loops.
- ❌ Avoid LINQ queries in update loops.
- ✅ Pre-allocate collections and reuse them with `.Clear()`.
- ✅ Use object pooling for frequently instantiated objects.
- ⚠️ A lambda allocates a delegate (a closure) when it captures an instance member or a local variable; one that
  touches only `static` members doesn't. In a genuinely hot path, avoid the lambda rather than making fields `static`
  to dodge the allocation. For UniTask waits, use the
  [closure-free overloads](UnityUniTaskInstructions.md#closure-free-overloads).

```csharp
// ❌ Bad - allocates every frame, and scans the whole scene
private void Update()
{
    var enemies = FindObjectsByType<Enemy>(FindObjectsSortMode.None); // Scene scan + array alloc
    var nearbyEnemies = new List<Enemy>();                            // Allocates list
    string status = $"Enemies: {enemies.Length}";                     // Allocates string

    foreach (var enemy in enemies.Where(e => e.IsAlive))              // LINQ allocates
    {
        nearbyEnemies.Add(enemy);
    }
}

// ✅ Good - zero allocations, no scene scan
// There is no non-allocating Find overload. The fix is not a better Find,
// it is not calling Find at all: have enemies register themselves.
private readonly List<Enemy> _activeEnemies = new(100);    // Populated by Enemy.OnEnable/OnDisable
private readonly List<Enemy> _nearbyEnemies = new(50);
private readonly StringBuilder _statusBuilder = new(64);

private void Update()
{
    _nearbyEnemies.Clear();                             // Reuses list

    for (int i = 0; i < _activeEnemies.Count; i++)
    {
        if (_activeEnemies[i].IsAlive)
        {
            _nearbyEnemies.Add(_activeEnemies[i]);
        }
    }

    _statusBuilder.Clear();                             // Reuses StringBuilder
    _statusBuilder.Append("Enemies: ").Append(_nearbyEnemies.Count);
}
```

## Caching Expensive Operations

- ✅ Cache results of expensive calculations outside Update when possible.
- ✅ Use dirty flags to recalculate only when state changes.
- ✅ Cache Transform, Rigidbody, and other component references in Awake().
- ❌ Avoid accessing `transform`, `gameObject`, or calling GetComponent every frame.

```csharp
// ❌ Bad - repeated property access and calculations
private void Update()
{
    Vector3 pos = transform.position;                   // Property access overhead
    Vector3 targetDir = (_target.transform.position - pos).normalized;
    float distance = Vector3.Distance(transform.position, _target.transform.position);
}

// ✅ Good - cached references and calculations
private Transform _transform;
private Transform _targetTransform;
private Vector3 _cachedTargetDirection;
private float _cachedDistance;
private bool _isDirty = true;

private void Awake()
{
    _transform = transform;                             // Cache once
    _targetTransform = _target.transform;
}

private void Update()
{
    if (_isDirty)
    {
        Vector3 offset = _targetTransform.position - _transform.position;
        _cachedDistance = offset.magnitude;
        _cachedTargetDirection = offset / _cachedDistance; // Avoid double sqrt
        _isDirty = false;
    }
}
```

## Throttling Update Logic

- ✅ Spread expensive work across multiple frames.
- ✅ Use time-based throttling for non-critical updates.
- ✅ For periodic checks, a UniTask loop with `UniTask.Delay` (see [Async and Coroutine Patterns](#async-and-coroutine-patterns))
  or a timer in `Update` as below.
- ⚠️ Be mindful of frame-rate dependent behavior when throttling.

```csharp
// ✅ Good - throttled updates
[SerializeField] private float _updateInterval = 0.1f;
private float _nextUpdateTime;

private void Update()
{
    if (Time.time < _nextUpdateTime) return;            // Skip until interval
    
    _nextUpdateTime = Time.time + _updateInterval;
    PerformExpensiveOperation();
}

// ✅ Good - staggered processing across frames
private int _currentIndex;
private const int ItemsPerFrame = 10;

private void Update()
{
    int endIndex = Mathf.Min(_currentIndex + ItemsPerFrame, _items.Count);
    
    for (int i = _currentIndex; i < endIndex; i++)
    {
        ProcessItem(_items[i]);
    }
    
    _currentIndex = endIndex >= _items.Count ? 0 : endIndex;
}
```

## Update Loop Discipline

- ✅ **Remove empty Unity lifecycle methods** (`Update`, `Start`, … left empty from a script template). Unity registers
  every defined lifecycle method for per-frame dispatch whether or not its body does anything.
- ✅ Tick plain C# logic that runs for a scope's whole lifetime — a Controller or service — with VContainer's
  **`ITickable`** (also `IFixedTickable`, `ILateTickable`), registered with `builder.RegisterEntryPoint<T>()`. Each scope
  runs all its tickables from one player-loop item instead of one `MonoBehaviour.Update` per object. The set of
  tickables is fixed when the scope is built, and ticking stops when the scope is disposed.
- ✅ Use **`Observable.EveryUpdate().Subscribe(...).AddTo(...)`** (R3) for per-frame work that starts and stops at
  runtime — "while this state is active". Each `Subscribe` allocates, so subscribe at setup, not every frame. Pass a
  frame provider (`UnityFrameProvider.FixedUpdate`, `PostLateUpdate`, …) to run at another point in the frame.
- ✅ For many spawned entities, use **one manager that loops over them** (ticked by either of the above) rather than
  one tickable or subscription per entity — the scalable version is a single loop over an array of their data.
- ⚠️ Reach for these when profiling shows `Update` dispatch overhead, not by default. Measure before and after.

```csharp
// Scope-lifetime logic: a plain C# Controller ticked by VContainer
public class EnemyWaveController : IEnemyWaveController, ITickable
{
    public void Tick()
    {
        // Advance the wave timer; runs every frame while this scene's scope exists
    }
}

// In the scene's LifetimeScope.Configure
builder.RegisterEntryPoint<EnemyWaveController>().As<IEnemyWaveController>();

// Runtime-lifetime work: per-frame only while this component is alive
private void Start()
{
    Observable.EveryUpdate()
        .Subscribe(_ => UpdateAim())
        .AddTo(this);
}
```

---

# Memory Management

## String Operations

- ❌ Never use string concatenation (`+`) in loops or frequent code paths.
- ❌ Avoid `string.Format()` in hot paths — it allocates.
- ✅ Use `StringBuilder` for building strings dynamically.
- ✅ Cache formatted strings when values don't change frequently.
- ✅ Use `string.Create()` or `Span<char>` for advanced zero-allocation scenarios.
- ✅ For UI text, `TMP_Text.SetText("{0}", value)` instead of `.text = value.ToString()`, and
  [ZString](https://github.com/Cysharp/ZString) (optional) for heavier formatting — see
  [Text](UnityUGUIInstructions.md#text) in the uGUI guide.
- ✅ Give a `StringBuilder` an initial capacity so it doesn't grow (and reallocate) as you append.

```csharp
// ❌ Bad - multiple allocations
private void UpdateUI()
{
    _scoreText.text = "Score: " + _score;                       // 2 allocations
    _healthText.text = string.Format("HP: {0}/{1}", _hp, _maxHp); // Allocates
}

// ✅ Good - TextMeshPro formats into its own buffer, and only when the value changed
private int _lastScore = -1;

private void UpdateUI()
{
    if (_score == _lastScore) return;

    _lastScore = _score;
    _scoreText.SetText("Score: {0}", _score);                  // No string allocated
    _healthText.SetText("HP: {0}/{1}", _hp, _maxHp);
}

// ✅ Good - a string that isn't going into a TMP label: build it only on change
private readonly StringBuilder _sb = new(32);
private string _cachedStatus;

private void RebuildStatus()
{
    _sb.Clear();
    _sb.Append("Wave ").Append(_wave);
    _cachedStatus = _sb.ToString();                             // Allocates once per change
}
```

## Collections and Allocations

- ✅ Initialize collections with expected capacity to avoid resizing.
- ✅ Prefer `List<T>.Clear()` over creating new lists.
- ✅ Use `CollectionPool<T>` or `ListPool<T>` from Unity's pooling utilities.
- ❌ Avoid `ToArray()`, `ToList()` in performance-critical code.
- ✅ Use `Span<T>` and `stackalloc` for temporary small arrays.

```csharp
// ❌ Bad - resizing allocations
private void ProcessEnemies()
{
    var enemies = new List<Enemy>();                    // No capacity, will resize
    // ... add many items
}

// ✅ Good - pre-sized capacity
private readonly List<Enemy> _enemies = new(100);      // Expected max capacity

// ✅ Good - using Unity's pooling
using UnityEngine.Pool;

private void ProcessWithPool()
{
    var tempList = ListPool<Enemy>.Get();
    try
    {
        // Use tempList...
    }
    finally
    {
        ListPool<Enemy>.Release(tempList);
    }
}

// ✅ Good - stackalloc for small temporary arrays (C# 7.2+)
private void ProcessSmallBatch()
{
    Span<int> indices = stackalloc int[8];             // No heap allocation
    // Use indices...
}
```

## Boxing and Unboxing

- ❌ Avoid passing value types to methods expecting `object`.
- ❌ Avoid storing value types in non-generic collections.
- ✅ Use generic collections (`List<int>` not `ArrayList`).
- ✅ Use generic methods and interfaces to avoid boxing.
- ⚠️ Watch for hidden boxing in string interpolation with value types.
- ⚠️ An unconstrained generic parameter compared with `Equals` falls back to `object.Equals` and boxes value types.
  Constrain it (`where T : IEquatable<T>`) so the non-boxing overload is called.

```csharp
// ❌ Bad - boxing occurs
object boxed = 42;                                      // Boxing
int unboxed = (int)boxed;                              // Unboxing

ArrayList oldList = new ArrayList();
oldList.Add(42);                                        // Boxing

Debug.Log($"Value: {myStruct}");                       // May box if no override

// ✅ Good - no boxing
List<int> genericList = new List<int>();
genericList.Add(42);                                    // No boxing

Debug.Log($"Value: {myInt}");                          // Primitives handled efficiently
```

## Loops Over Collections

For **hot paths only** — cold code (menus, setup) keeps `foreach` and LINQ for clarity. Figures below are from the
[Unity Performance Tuning Bible](https://cyberagentgameentertainment.github.io/UnityPerformanceTuningBible/en/)'s benchmarks on large data sets; read the ordering, not the exact multiple.

- ✅ Iterate a large, hot collection as an **array** where the size allows it — arrays iterated roughly 2.3× faster
  than `List<T>` in those benchmarks.
- ✅ Over a `List<T>`, prefer `for` with `Count` cached in a local to `foreach`: it skips the enumerator and the
  per-iteration `Count` read.

```csharp
// Hot path: cached count, indexer access
int count = _enemies.Count;
for (int i = 0; i < count; i++)
{
    _enemies[i].Tick();
}
```

## Unity Objects You Own

Some Unity objects are native resources that the garbage collector never frees. If your code creates them, your code
destroys them.

- ✅ `Texture2D`, `Sprite`, `Material`, `Mesh` and `RenderTexture` created at runtime (`new`, `Sprite.Create`,
  `Instantiate`) → `Destroy()` them when finished. A `RenderTexture` also needs `Release()`.
- ✅ A `PlayableGraph` → `graph.Destroy()`.
- ⚠️ `Renderer.material` and `MeshFilter.mesh` **clone** on first access per object. Cache the reference and destroy
  the clone in `OnDestroy` — see [Rendering Considerations](#rendering-considerations).
- ✅ A texture filled from code: once the pixels are final, call `Apply(updateMipmaps, makeNoLongerReadable: true)`.
  A readable texture keeps a CPU copy in main memory alongside the GPU copy; this drops it.

```csharp
private Texture2D _minimapTexture;

private void BuildMinimap(int size)
{
    _minimapTexture = new Texture2D(size, size, TextureFormat.RGBA32, mipChain: false);
    FillPixels(_minimapTexture);
    _minimapTexture.Apply(updateMipmaps: false, makeNoLongerReadable: true);  // Drop the CPU copy
}

private void OnDestroy()
{
    if (_minimapTexture != null)
    {
        Destroy(_minimapTexture);   // Not garbage-collected
    }
}
```

---

# Object Pooling

- ✅ Use `UnityEngine.Pool.ObjectPool<T>` for frequently spawned objects (bullets, particles, UI elements).
- ✅ Implement `IDisposable` pattern or use `actionOnRelease` to reset object state.
- ✅ Set appropriate `defaultCapacity` and `maxSize` based on expected usage.
- ❌ Don't use `Instantiate`/`Destroy` for objects spawned more than a few times per second.
- ⚠️ Remember to return objects to the pool — leaked pooled objects defeat the purpose.

```csharp
using UnityEngine;
using UnityEngine.Pool;

public class ProjectilePool : MonoBehaviour
{
    [SerializeField] private Projectile _prefab;
    [SerializeField] private int _defaultCapacity = 20;
    [SerializeField] private int _maxSize = 100;
    
    private ObjectPool<Projectile> _pool;

    private void Awake()
    {
        _pool = new ObjectPool<Projectile>(
            createFunc: CreateProjectile,
            actionOnGet: OnGetFromPool,
            actionOnRelease: OnReturnToPool,
            actionOnDestroy: OnDestroyPooled,
            collectionCheck: false,                     // Disable in release for perf
            defaultCapacity: _defaultCapacity,
            maxSize: _maxSize
        );
    }

    private Projectile CreateProjectile()
    {
        var proj = Instantiate(_prefab);
        proj.SetPool(_pool);                            // Give projectile pool reference
        return proj;
    }

    private void OnGetFromPool(Projectile proj)
    {
        proj.gameObject.SetActive(true);
        proj.ResetState();
    }

    private void OnReturnToPool(Projectile proj)
    {
        proj.gameObject.SetActive(false);
    }

    private void OnDestroyPooled(Projectile proj)
    {
        Destroy(proj.gameObject);
    }

    public Projectile Get() => _pool.Get();
    public void Return(Projectile proj) => _pool.Release(proj);
}
```

---

# Physics Optimization

**Cost only.** Which physics API is *correct* — movement, collision callbacks, collider choice,
tunnelling — lives in [Physics](UnityPhysicsInstructions.md).

- ✅ Use layer masks to limit physics queries to relevant layers.
- ✅ Cache `LayerMask` values — don't call `LayerMask.GetMask()` every frame.
- ✅ Use non-allocating physics methods: `Physics.RaycastNonAlloc`, `Physics.OverlapSphereNonAlloc`.
- ✅ Prefer simple colliders. Rough cost, cheapest first: sphere < capsule < box < mesh. A capsule approximating a
  character can often be a sphere if height doesn't matter to the gameplay.
- ✅ Prefer `Physics.Raycast` to shape casts (`SphereCast`, `BoxCast`, …) when a line is enough.
- ✅ Keep `Physics.reuseCollisionCallbacks` on (the default) — otherwise every `OnCollision*` call allocates a
  `Collision`. See [What actually fires](UnityPhysicsInstructions.md#what-actually-fires) for the catch.
- ✅ Turn the simulation off where no gameplay needs it — see
  [Simulation control](UnityPhysicsInstructions.md#simulation-control) — and let idle bodies
  [sleep](UnityPhysicsInstructions.md#sleeping).
- ❌ Avoid physics queries in Update — use FixedUpdate or throttle them.
- ✅ Configure the Physics collision matrix to disable unnecessary layer interactions.

```csharp
// ❌ Bad - allocating physics query every frame
private void Update()
{
    Collider[] hits = Physics.OverlapSphere(transform.position, _radius);
    foreach (var hit in hits)
    {
        // Process...
    }
}

// ✅ Good - non-allocating with cached arrays and layer mask
private readonly Collider[] _hitBuffer = new Collider[32];
private LayerMask _enemyLayer;

private void Awake()
{
    _enemyLayer = LayerMask.GetMask("Enemy");           // Cache layer mask
}

private void FixedUpdate()
{
    int hitCount = Physics.OverlapSphereNonAlloc(
        transform.position, 
        _radius, 
        _hitBuffer,
        _enemyLayer                                      // Only check enemy layer
    );
    
    for (int i = 0; i < hitCount; i++)
    {
        ProcessHit(_hitBuffer[i]);
    }
}
```

## Raycast Optimization

- ✅ Use `Physics.Raycast` with `maxDistance` parameter to limit range.
- ✅ Use `QueryTriggerInteraction.Ignore` if you don't need trigger colliders.
- ✅ For multiple raycasts, consider `Physics.RaycastCommand` with Jobs for batching.

```csharp
// ✅ Good - optimized raycast
private RaycastHit _hitInfo;
private const float MaxRayDistance = 100f;

private bool CheckLineOfSight(Vector3 origin, Vector3 direction)
{
    return Physics.Raycast(
        origin,
        direction,
        out _hitInfo,
        MaxRayDistance,
        _lineOfSightMask,
        QueryTriggerInteraction.Ignore
    );
}
```

---

# Rendering Considerations

- ⚠️ Accessing `.material` creates a material instance — use `.sharedMaterial` when possible.
- ⚠️ **The instance `.material` creates is a leak.** Unity does not destroy it with the GameObject.
  If you take `.material`, you own it: `Destroy(_renderer.material)` in `OnDestroy`. On pooled
  objects that touch `.material` per spawn, this is a steady leak.
- ⚠️ Writing to `.sharedMaterial` edits the material **asset**. In the Editor the change persists
  after exiting play mode, and every renderer using that material changes with it.
- ✅ Batch material property changes using `MaterialPropertyBlock`.
- ✅ Use `Renderer.GetPropertyBlock` / `SetPropertyBlock` for per-instance changes.
- ⚠️ A `MaterialPropertyBlock` breaks SRP Batcher compatibility for that renderer. It's still the
  right tool for per-instance tints, but it isn't free — see [SRP Batcher](#srp-batcher). Setting it on very large
  instance counts every frame has its own CPU cost too.
- ❌ Avoid changing materials at runtime unless necessary.
- ✅ Use GPU instancing for many similar objects.
- ✅ Prefer `LocalKeyword` over global keywords when toggling shader features on one material.
  Toggling keywords per frame forces variant switches and stalls render state.
- ⚠️ `volume.profile` **clones** the VolumeProfile asset and leaks it the same way `.material` does.
  Use `volume.sharedProfile` to read, and only take `.profile` when you genuinely need a per-volume
  copy — then destroy it.
- ✅ Setting a Volume override from script needs both halves: `vignette.intensity.value = 0.5f;`
  **and** `vignette.intensity.overrideState = true;`. Without the second line the blend ignores it.
- ✅ Subscribe to `RenderPipelineManager.beginCameraRendering` in `OnEnable` and unsubscribe in
  `OnDisable`, like any other event. It fires per camera per frame — keep the body cheap.
- ⚠️ Every URP Overlay camera in a stack pays full culling, sorting and pass setup. Reach for
  sorting layers or a render pass before adding a camera.

```csharp
// ❌ Bad - creates material instance per object
private void Start()
{
    GetComponent<Renderer>().material.color = Color.red; // Creates instance!
}

// ✅ Good - uses MaterialPropertyBlock (no allocation after first call)
private static readonly int BaseColorId = Shader.PropertyToID("_BaseColor");
private MaterialPropertyBlock _propertyBlock;
private Renderer _renderer;

private void Awake()
{
    _renderer = GetComponent<Renderer>();
    _propertyBlock = new MaterialPropertyBlock();
}

private void SetColor(Color color)
{
    _renderer.GetPropertyBlock(_propertyBlock);
    _propertyBlock.SetColor(BaseColorId, color);
    _renderer.SetPropertyBlock(_propertyBlock);
}
```

## Shader Property IDs

- ✅ Cache shader property IDs with `Shader.PropertyToID()`.
- ❌ Never use string-based property access in update loops.
- ⚠️ URP shaders use different property names from the Built-in pipeline: `_BaseColor` and
  `_BaseMap`, not `_Color` and `_MainTex`. A wrong name doesn't error — the property is silently
  ignored.

```csharp
// ❌ Bad - string lookup every call
_material.SetFloat("_Intensity", value);

// ✅ Good - cached ID
private static readonly int IntensityId = Shader.PropertyToID("_Intensity");

private void UpdateShader(float value)
{
    _material.SetFloat(IntensityId, value);
}
```

---

# GetComponent and Find Operations

- ❌ **Never call GetComponent in Update** — cache in Awake/Start.
- ❌ Avoid `FindAnyObjectByType`, `FindObjectsByType` at runtime — they are O(n) scene scans.
- ℹ️ `FindObjectOfType` / `FindObjectsOfType` are deprecated (since 2023.1) — replace with
  `FindAnyObjectByType` / `FindObjectsByType`. On 6000.6+, also avoid `FindFirstObjectByType` and
  any `FindObjectsByType` overload taking a `FindObjectsSortMode` — both became Obsolete in 6000.6,
  in favor of `FindAnyObjectByType` and the parameterless `FindObjectsByType<T>()`. That
  parameterless overload doesn't exist before 6000.6 — check
  `UnityCustomInstructions/UnityTechStack.md` for the project's version, and keep
  `FindObjectsByType<T>(FindObjectsSortMode.None)` on 6000.0–6000.5.
- ❌ Avoid `GameObject.Find` — string-based, searches entire hierarchy.
- ✅ Use `[SerializeField]` to assign references in the Inspector.
- ✅ Use `TryGetComponent` for null-safe lookups (slightly faster than GetComponent + null check).
- ✅ Inject cross-system references with VContainer — see
  [UnityArchitectureInstructions.md](UnityArchitectureInstructions.md#dependency-injection-vcontainer). Never a
  static locator.

```csharp
// ❌ Bad - expensive lookups every frame
private void Update()
{
    var rb = GetComponent<Rigidbody>();                 // Lookup every frame
    var player = FindAnyObjectByType<Player>();         // Scene scan every frame
    var enemy = GameObject.Find("Enemy");              // String search every frame
}

// ✅ Good - cached references
[SerializeField] private Rigidbody _rigidbody;
[SerializeField] private Player _player;
private Transform _cachedTransform;

private void Awake()
{
    _cachedTransform = transform;
    
    // Cache if not assigned in Inspector
    if (_rigidbody == null)
    {
        TryGetComponent(out _rigidbody);
    }
}

private void Update()
{
    // Use cached references
    _rigidbody.AddForce(Vector3.up);
}
```

## RequireComponent Pattern

- ✅ Use `[RequireComponent]` to ensure dependencies exist and enable caching confidence.

```csharp
[RequireComponent(typeof(Rigidbody))]
[RequireComponent(typeof(Collider))]
public class PhysicsBody : MonoBehaviour
{
    private Rigidbody _rigidbody;
    private Collider _collider;

    private void Awake()
    {
        // Safe to cache — RequireComponent guarantees existence
        _rigidbody = GetComponent<Rigidbody>();
        _collider = GetComponent<Collider>();
    }
}
```

---

# LINQ and Delegates

- ❌ **Never use LINQ in Update loops** — most LINQ methods allocate.
- ❌ Avoid lambda expressions in hot paths — they can allocate closures.
- ✅ Use explicit loops instead of LINQ for performance-critical code.
- ✅ Cache delegates when subscribing to events repeatedly.
- ⚠️ LINQ is fine for initialization, editor code, or infrequent operations.
- ℹ️ The [Unity Performance Tuning Bible](https://cyberagentgameentertainment.github.io/UnityPerformanceTuningBible/en/)'s benchmark measured LINQ at roughly 19× slower than an equivalent manual loop over a large data
  set, on top of the iterator allocations.

```csharp
// ❌ Bad - LINQ allocations in Update
private void Update()
{
    var activeEnemies = _enemies.Where(e => e.IsActive).ToList();
    var closestEnemy = _enemies.OrderBy(e => e.Distance).FirstOrDefault();
}

// ✅ Good - explicit loops, no allocations
private Enemy _closestEnemy;
private readonly List<Enemy> _activeEnemies = new(50);

private void Update()
{
    _activeEnemies.Clear();
    float minDistance = float.MaxValue;
    _closestEnemy = null;
    
    for (int i = 0; i < _enemies.Count; i++)
    {
        var enemy = _enemies[i];
        if (enemy.IsActive)
        {
            _activeEnemies.Add(enemy);
            
            if (enemy.Distance < minDistance)
            {
                minDistance = enemy.Distance;
                _closestEnemy = enemy;
            }
        }
    }
}
```

## Delegate Caching

```csharp
// ❌ Bad - the lambda captures `this`, allocating a closure, and can't be unsubscribed
private void OnEnable()
{
    _jumpAction.performed += context => Jump();
}

// ✅ Good - method group, with a matching unsubscribe
private void OnEnable()
{
    _jumpAction.performed += HandleJumpPerformed;
}

private void OnDisable()
{
    _jumpAction.performed -= HandleJumpPerformed;
}

private void HandleJumpPerformed(InputAction.CallbackContext context)
{
    Jump();
}
```

---

# Async and Coroutine Patterns

- ✅ Use UniTask for async gameplay code — see [UnityUniTaskInstructions.md](UnityUniTaskInstructions.md). It doesn't
  capture `SynchronizationContext`/`ExecutionContext` on each `await`, a per-await cost that `Task` pays.
- ✅ Don't mark a method `async` if it usually completes synchronously — the state machine is generated either way.
  Split the synchronous fast path from the genuinely asynchronous one (see
  [the UniTask guide](UnityUniTaskInstructions.md#dont-mark-a-method-async-if-it-doesnt-need-to-be)).
- ✅ Pass a destroy token (`this.GetCancellationTokenOnDestroy()`): after `Destroy()` the await throws
  `OperationCanceledException` instead of resuming, so no `this == null` checks are needed.
- ✅ In the rare legitimate coroutine, cache `WaitForSeconds` objects; never create one per loop iteration.
- ✅ Use `WaitForSecondsRealtime` (coroutines) or `UniTask.Delay(..., ignoreTimeScale: true)` for unscaled time.

```csharp
// ❌ Bad - allocates WaitForSeconds every iteration
private IEnumerator PollRoutine()
{
    while (true)
    {
        yield return new WaitForSeconds(0.1f);         // Allocates each loop
        DoSomething();
    }
}

// ✅ Good - cached wait object
private static readonly WaitForSeconds ShortWait = new(0.1f);

private IEnumerator PollRoutine()
{
    while (true)
    {
        yield return ShortWait;                         // Reuses cached object
        DoSomething();
    }
}

// ✅ Better - UniTask loop, cancelled when the object is destroyed
private async UniTaskVoid TaskPoll(CancellationToken token)
{
    try
    {
        while (true)
        {
            await UniTask.Delay(100, cancellationToken: token);
            DoSomething();
        }
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

// Started with: TaskPoll(this.GetCancellationTokenOnDestroy()).Forget();
```

---

# Data Structure Selection

- ✅ Use `Dictionary<K,V>` for O(1) lookups by key.
- ✅ Use `HashSet<T>` for O(1) contains checks.
- ✅ Use `List<T>` when order matters and you iterate frequently.
- ✅ Use arrays for fixed-size, frequently-accessed data.
- ✅ Use `Queue<T>` for FIFO operations.
- ✅ Use `Stack<T>` for LIFO operations (undo systems, state history).
- ⚠️ Consider `NativeArray<T>` for Jobs/Burst compatibility.

```csharp
// Choose the right data structure for the operation

// O(1) lookup by ID
private Dictionary<int, Enemy> _enemyById = new();

// O(1) membership check
private HashSet<int> _processedIds = new();

// Ordered iteration, dynamic size
private List<Enemy> _activeEnemies = new();

// Fixed size, frequent access
private Enemy[] _enemyPool = new Enemy[100];

// Command queue
private Queue<ICommand> _commandQueue = new();

// Undo stack
private Stack<ICommand> _undoStack = new();
```

---

# Unity API Best Practices

## Transform Operations

- ✅ Batch transform changes — multiple SetPosition calls are inefficient.
- ✅ Use `Transform.SetPositionAndRotation()` when setting both.
- ❌ Avoid modifying individual components of position/rotation separately.

```csharp
// ❌ Bad - multiple transform operations
transform.position = newPosition;
transform.rotation = newRotation;

// ✅ Good - single combined operation
transform.SetPositionAndRotation(newPosition, newRotation);

// ❌ Bad - modifying individual components
transform.position = new Vector3(x, transform.position.y, transform.position.z);

// ✅ Good - set complete vector
Vector3 pos = transform.position;
pos.x = x;
transform.position = pos;
```

## CompareTag vs ==

- ✅ Use `CompareTag()` instead of `==` for tag comparison — no allocation.

```csharp
// ❌ Bad - string allocation for comparison
if (other.gameObject.tag == "Player")

// ✅ Good - no allocation
if (other.CompareTag("Player"))
```

## Camera.main

- ❌ Avoid `Camera.main` in Update — it performs a FindGameObjectWithTag internally.
- ✅ Cache the main camera reference.

```csharp
// ❌ Bad - lookup every frame
private void Update()
{
    Vector3 screenPos = Camera.main.WorldToScreenPoint(transform.position);
}

// ✅ Good - cached reference
private Camera _mainCamera;

private void Awake()
{
    _mainCamera = Camera.main;
}

private void Update()
{
    Vector3 screenPos = _mainCamera.WorldToScreenPoint(transform.position);
}
```

---

# Profiling Markers

- ✅ Add profiler markers to identify expensive methods in the Unity Profiler.
- ✅ Use `ProfilerMarker` for lightweight, zero-allocation profiling.
- ✅ Remove or conditionally compile profiling code for release builds.

```csharp
using Unity.Profiling;

public class PerformanceCriticalSystem : MonoBehaviour
{
    private static readonly ProfilerMarker UpdateMarker = 
        new ProfilerMarker("PerformanceCriticalSystem.Update");
    
    private static readonly ProfilerMarker ProcessEnemiesMarker = 
        new ProfilerMarker("PerformanceCriticalSystem.ProcessEnemies");

    private void Update()
    {
        using (UpdateMarker.Auto())
        {
            ProcessEnemies();
            UpdateUI();
        }
    }

    private void ProcessEnemies()
    {
        using (ProcessEnemiesMarker.Auto())
        {
            // Expensive processing...
        }
    }
}
```

---

# Garbage Collection

- ℹ️ Unity 6 uses the Boehm collector with **incremental mode on by default**. It splits collection
  across frames, turning one long spike into several short ones. Total work is unchanged.
- ⚠️ **The managed heap never shrinks.** Once a spike grows it, that memory is reserved for the
  process lifetime. A single sloppy loading screen permanently raises your memory floor.
- ✅ The goal is fewer *allocations*, not fewer collections. Everything in
  [Memory Management](#memory-management) feeds this.
- ✅ For a section that must not hitch — a cutscene, a ranked round — you can suspend collection
  entirely, provided allocation during that window is bounded.

```csharp
using UnityEngine.Scripting;

private void BeginNoHitchSection()
{
    // The heap grows instead of collecting. Only safe if allocation is bounded.
    GarbageCollector.GCMode = GarbageCollector.Mode.Disabled;
}

private void EndNoHitchSection()
{
    GarbageCollector.GCMode = GarbageCollector.Mode.Enabled;
    GC.Collect();   // Pay the cost at a moment you choose
}
```

- ℹ️ Asset memory usually dwarfs managed memory. Before optimizing allocations, confirm that's
  actually where your budget is going — see
  [UnityAssetsAndMemoryInstructions.md](UnityAssetsAndMemoryInstructions.md).

---

# Jobs and Burst

- ✅ Use the Job System with Burst for **data-parallel, math-heavy** work: pathfinding grids, flocking,
  procedural mesh generation, large-scale simulation.
- ❌ Don't job-ify light work. Scheduling has overhead; a job over 50 elements is slower than a loop.
- ✅ Jobs may only touch blittable types and `NativeContainer`s. No managed objects, no
  `GameObject`, no `Transform` (except via `TransformAccessArray`).
- ⚠️ Every `NativeArray` you allocate must be disposed, or the Editor logs a leak on exit.
- ✅ Schedule early in the frame, complete late — that's the window where parallelism actually buys
  you anything.

```csharp
using Unity.Burst;
using Unity.Collections;
using Unity.Jobs;
using Unity.Mathematics;

[BurstCompile]
public struct MoveTowardsJob : IJobParallelFor
{
    [ReadOnly] public NativeArray<float3> Targets;
    [ReadOnly] public float DeltaTime;
    [ReadOnly] public float Speed;

    public NativeArray<float3> Positions;

    public void Execute(int index)
    {
        float3 offset = Targets[index] - Positions[index];
        float distance = math.length(offset);

        if (distance < 0.001f) return;

        Positions[index] += (offset / distance) * Speed * DeltaTime;
    }
}

// Usage
private void Update()
{
    var job = new MoveTowardsJob
    {
        Targets = _targets,
        Positions = _positions,
        DeltaTime = Time.deltaTime,
        Speed = _speed
    };

    // 64 = batch size; tune it, don't guess once and forget
    _handle = job.Schedule(_positions.Length, 64);
}

private void LateUpdate()
{
    _handle.Complete();    // Complete as late as possible
}

private void OnDestroy()
{
    // NativeArrays are not garbage collected
    if (_positions.IsCreated) _positions.Dispose();
    if (_targets.IsCreated) _targets.Dispose();
}
```

- ✅ Enable **Burst AOT** and **Synchronous Compilation** so you don't measure the JIT warm-up as a
  first-frame stutter.

---

# Rendering Performance

## SRP Batcher

- ✅ The SRP Batcher is on by default in URP and batches by **shader variant**, not by material. Many
  materials sharing one shader still batch.
- ⚠️ A shader is SRP Batcher compatible only if all its per-material properties live in a
  `UnityPerMaterial` CBUFFER. Shader Graph does this automatically; hand-written shaders often don't.
- ✅ Check compatibility in the Inspector for the shader asset — it says "SRP Batcher: compatible" or
  gives the reason it isn't.
- ❌ `MaterialPropertyBlock` **breaks SRP Batcher compatibility** for that renderer. It's still the
  right answer for GPU instancing and for occasional per-instance tweaks, but don't reach for it by
  reflex — see [Rendering Considerations](#rendering-considerations).

## GPU Resident Drawer and occlusion culling

- ✅ Unity 6's **GPU Resident Drawer** moves rendering of static geometry onto the GPU, cutting CPU
  draw-call submission dramatically for large scenes. Enable it in the URP asset.
- ⚠️ It requires objects to be marked **Batching Static** and works only with SRP Batcher-compatible
  shaders.
- ✅ **GPU Occlusion Culling** pairs with it, skipping objects hidden behind others. Biggest wins are
  in dense interiors and cities; open vistas benefit less.
- ✅ Measure both on and off. On scenes with few objects the setup cost can exceed the saving.

## Frame pacing

- ✅ Set `Application.targetFrameRate` explicitly. The default on mobile is 30 and on desktop is
  uncapped, which burns battery and generates heat for frames nobody sees.
- ⚠️ `QualitySettings.vSyncCount` overrides `targetFrameRate` when it's non-zero. Set vSync to 0 if
  you want the target to take effect.
- ✅ A stable 30 fps feels better than an average 45 that swings. Optimize the worst frame, not the
  mean.

```csharp
[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
private static void ConfigureFrameRate()
{
    QualitySettings.vSyncCount = 0;        // Otherwise targetFrameRate is ignored
    Application.targetFrameRate = 60;
}
```

## Batching settings

- ✅ **SRP Batcher** — on by default in URP (Rendering section of the URP Asset); see [SRP Batcher](#srp-batcher).
- ✅ **Static Batching** — for non-moving objects sharing a material: the Player Settings toggle plus the object's
  **Batching Static** flag. No runtime CPU cost, but the combined mesh costs memory.
- ❌ **Dynamic Batching** — leave it off on URP. It has a steady CPU cost and the SRP Batcher supersedes it.
- ✅ **GPU Instancing** — **Enable Instancing** on the material, for many copies of one mesh and material (foliage,
  crowds). The shader must support instancing to benefit.
- ✅ Fewer distinct materials and textures per object means fewer set-pass calls — a CPU cost, not a GPU one.
- ✅ The **Frame Debugger** says why each draw call didn't batch with the previous one. Check it before changing any
  of the settings above.

## Culling

- ℹ️ Frustum culling is automatic. A large mesh is culled only as a whole, so splitting it lets more of it be culled.
- ℹ️ Backface culling is a shader setting (`Cull Back` / `Front` / `Off`), not an Inspector toggle.
- ✅ **Baked Occlusion Culling**: mark objects **Occluder Static** and/or **Occludee Static**, then bake in
  **Window → Rendering → Occlusion Culling**. It adds a CPU culling cost of its own, so measure it on and off. For
  large scenes on URP, also compare [GPU Occlusion Culling](#gpu-resident-drawer-and-occlusion-culling).

## Shadows and lighting

- ✅ Turn **Cast Shadows** off on renderers whose shadow nobody sees — it removes them from the shadow pass.
- ✅ On URP, shadow cost is set on the URP Asset's **Shadows** section — **Max Distance**, **Cascade Count**, **Soft
  Shadows** — with the main light's **Shadow Resolution** in its **Lighting** section. Lower distance and cascades are
  the biggest levers. (Quality Settings → Shadows applies to the Built-in pipeline, not URP.)
- ✅ Bake lighting (**Baked** or **Mixed** light mode) wherever the light and what it lights don't move. Baked light
  costs almost nothing at runtime.

## LOD and mipmap streaming

- ⚠️ A **LOD Group** lowers render cost with distance, but every LOD level's mesh stays in memory — it trades memory
  for render time.
- ✅ **Mipmap streaming** (called texture streaming in older versions) loads only the mip levels the camera needs:
  enable it in Quality Settings → Textures with a **Memory Budget**, and enable streaming on each texture in its
  import settings.

## Render resolution

- ✅ On URP, the URP Asset's **Render Scale** (Quality section) lowers the 3D render resolution while
  Screen Space – Overlay UI stays at native resolution — usually the cheapest fill-rate win on mobile.
- ℹ️ Player Settings → Resolution Scaling Mode **Fixed DPI** sets the device resolution from a target DPI.
  `Screen.SetResolution` changes it at runtime, on a device only — not in the Editor.

## Particle systems

- ✅ Cap particle counts: the main module's **Max Particles**, and the Emission module's **Rate over Time** and
  **Bursts** counts.
- ⚠️ Audit **Sub Emitters** — each one can spawn a whole secondary system per birth or death, pushing the count far
  past the main caps.
- ✅ Keep the **Noise** module's **Quality** at Low unless higher is needed, and disable the module when unused.
- ⚠️ Semi-transparent particles can't skip pixels already covered, so stacked or screen-filling layers multiply
  fill-rate cost (overdraw). Check with the Scene view's Overdraw mode.

## Animator cost

- ⚠️ Each `Animator` has a fixed per-frame cost even when the state machine is idle. Hundreds of them
  add up before any animation actually plays.
- ✅ Set **Culling Mode** to `Cull Update Transforms` or `Cull Completely` so offscreen characters
  stop evaluating.
  - ⚠️ `Cull Update Transforms` keeps the state machine running but skips transform writes, so a transform-driven
    effect can visibly jump when the character comes back into view.
  - ⚠️ `Cull Completely` stops the state machine offscreen, so root motion that should walk a character back into
    view never advances.
- ✅ For a lower update rate than Culling Mode offers (e.g. distant but visible characters), disable the `Animator`
  component and call `animator.Update(deltaTime)` yourself at the rate you want.
- ✅ For simple, non-blended motion — a rotating pickup, a bobbing platform — plain code in `Update`
  is far cheaper than an Animator.
- ✅ Disable the Animator component outright when a character is idle and offscreen.
- ℹ️ Never put an Animator on uGUI elements. See
  [UnityUGUIInstructions.md](UnityUGUIInstructions.md#never-animate-ui-with-an-animator).

## UI

UI is a frequent and frequently-missed source of CPU cost. Canvas rebuilds don't show up where people
usually look.

- ℹ️ uGUI: see [UnityUGUIInstructions.md](UnityUGUIInstructions.md#performance-rules) — Canvas
  granularity, raycast targets, and layout groups are the big three.
- ℹ️ UI Toolkit: see [UnityUIToolkitInstructions.md](UnityUIToolkitInstructions.md).

---

# Profiling Workflow

Markers are useless without a method for reading them.

## Where to look, in order

1. **Profiler → CPU Usage**, Timeline view. Find the spike, not the average.
2. Check whether you're **CPU or GPU bound** — Unity 6.3's Highlights module details pane splits this
   out directly. Optimizing the wrong one wastes days.
3. Expand `PlayerLoop` and follow the largest child down.
4. For rendering, switch to the **Frame Debugger** and step through draw calls.
5. For memory, take a **Memory Profiler** snapshot and diff two of them.

## Rules that save time

- ⚠️ **Profile a development build on the target device.** Editor numbers are not representative —
  the Editor holds extra copies of assets and adds its own overhead everywhere.
- ⚠️ **Deep Profile distorts relative cost.** It adds instrumentation to every method, so small
  frequently-called methods look far worse than they are. Use it to find *where*, then turn it off
  and measure the real cost with a `ProfilerMarker`.
- ✅ Profile with **Autoconnect Profiler** or connect over the network. `Development Build` alone
  isn't enough to get the connection.
- ✅ Record a few hundred frames and look at the distribution. One bad frame in 200 is a hitch players
  will notice, and it averages away to nothing.
- ✅ Unity 6.3's **Captures List** stores Profiler sessions in the project so you can reopen a
  before/after pair without re-importing.
- ✅ Change one thing at a time and re-measure. Two simultaneous optimizations can cancel out and look
  like no change.

| Tool | Use for |
|---|---|
| Profiler → CPU | Frame time, spikes, which subsystem |
| Profiler → Memory | Managed heap, GC allocation rate |
| Memory Profiler package | Native memory, what holds a reference, leak diffs (**Compare Snapshots**) |
| Frame Debugger | Draw calls, batch breaks, render order |
| Profile Analyzer package | Comparing two capture sets statistically |
| Highlights module (6.3) | Fast CPU/GPU-bound triage |
| Heap Explorer (optional, open source) | A lighter alternative for managed-memory and reference tracing |
| Xcode Instruments (iOS) | Time Profiler and Allocations down to the call site; GPU Frame Capture for shader stages |
| Android Studio Profiler | CPU call-stack sampling and heap dumps on a running development build |
| RenderDoc (Windows, Linux, Android — not iOS) | GPU frame capture with per-pixel draw history, for overdraw |

- ✅ In **CPU Usage**, the **Hierarchy** view sorted by **GC Alloc** finds allocation sources; **Raw Hierarchy**
  keeps repeated calls as separate rows when one of several identical calls is the slow one; **Timeline** shows all
  threads across the frame.
- ✅ In **Memory**, the simple view shows growth live (heap, reserved memory, object counts) — the first sign of a
  leak. A detailed sample's **Referenced By** column shows what still holds an object.
- ✅ With a live connection to the Editor (e.g. through MCP tooling), route a suspicion to the matching tool before
  changing code: an allocation spike to CPU Usage sorted by GC Alloc, a leak or duplicate asset to Memory Profiler's
  Compare Snapshots, a batching problem to the Frame Debugger.

## Profiling checklist

Before merging performance-sensitive code, check the Profiler for:

- **GC.Alloc** in the relevant frames — trace it to `new`, a closure, LINQ, boxing, or string building.
- **Repeated identical Unity API calls** in one frame (`GetComponent`, `.tag`, `.name`, `transform`,
  `Camera.main`) that should be cached.
- For CPU-bound rendering: draw-call and set-pass counts, and SRP Batcher compatibility of the shaders involved.
- A before/after comparison in Profile Analyzer rather than one frame of each.
- Big-O reasoning guides the choice of algorithm; it doesn't replace measuring at the real data size.

---

# Common Anti-Patterns

## Anti-Pattern Checklist for Code Review

When reviewing Unity code, flag these patterns:

| Anti-Pattern | Impact | Solution |
|--------------|--------|----------|
| `GetComponent` in Update | High | Cache in Awake |
| `FindAnyObjectByType`/`FindObjectsByType` at runtime | High | Use references or events |
| `new List<T>()` in Update | High | Pre-allocate and Clear() |
| String concatenation in loops | Medium | Use StringBuilder |
| `Camera.main` in Update | Medium | Cache reference |
| LINQ in Update | Medium | Use explicit loops |
| `Physics.Raycast` every frame | Medium | Throttle or use FixedUpdate |
| `material` instead of `sharedMaterial` | Medium | Use MaterialPropertyBlock |
| Lambda in event subscription | Low | Use method group |
| `new WaitForSeconds` in coroutine loop | Low | Cache wait object |
| `Animator` on a uGUI element | High | Dirties the canvas every frame — use code or a tween |
| Monolithic Canvas for the whole UI | High | Split by update frequency |
| `Raycast Target` on decorative graphics | Medium | Turn it off |
| `NativeArray` never disposed | Medium | Dispose in `OnDestroy` |
| Deep Profile used to compare costs | Medium | Distorts relative cost — use `ProfilerMarker` |
| Runtime-created `Texture2D`/`Material`/`Mesh` never destroyed | Medium | `Destroy()` it when finished |
| Empty `Update`/`Start` left in a script | Low | Delete it |
| Dynamic Batching enabled on URP | Low | Turn it off; rely on the SRP Batcher |

## Code Smell Detection

Look for these patterns that indicate potential issues:

```csharp
// 🔴 Red flags in Update methods
void Update()
{
    GetComponent<T>()                    // 🔴 Uncached lookup
    FindAnyObjectByType<T>()             // 🔴 Scene scan
    new List<T>()                        // 🔴 Allocation
    new T[]                              // 🔴 Allocation
    string + string                      // 🔴 String allocation
    $"interpolated {value}"              // 🔴 String allocation
    .Where() .Select() .ToList()         // 🔴 LINQ allocation
    Camera.main                          // 🔴 Uncached lookup
    GameObject.Find()                    // 🔴 String search
    Physics.OverlapSphere()              // ⚠️ Allocating version
}

// 🟢 Preferred patterns
void Update()
{
    _cachedComponent                     // 🟢 Cached reference
    _cachedList.Clear()                  // 🟢 Reused collection
    _stringBuilder.Clear().Append()      // 🟢 Reused builder
    for (int i = 0; i < count; i++)      // 🟢 Explicit loop
    _cachedCamera                        // 🟢 Cached reference
    Physics.OverlapSphereNonAlloc()      // 🟢 Non-allocating
}
```

---

# Troubleshooting

**The Profiler shows a spike but the call stack is all `PlayerLoop` with nothing underneath.**
The cost is in native code the Profiler doesn't break down by default. Enable **Deep Profile** to
locate it, then turn it off and add a `ProfilerMarker` around the suspect region to measure it
honestly.

**Deep Profile says a method is expensive; removing it changes nothing.**
Deep Profile instruments every method call, so cost is dominated by call *count*, not real work. A
tiny method called 10,000 times looks catastrophic. Never compare costs with Deep Profile on.

**Frame rate is fine on average but the game feels bad.**
Look at the frame time graph, not the average. One 60 ms frame per second is invisible in a mean and
extremely visible to a player. Sort by worst frame.

**GC spikes with no obvious allocation in your code.**
Common hidden sources: `foreach` over a non-generic collection (boxes the enumerator), a lambda that
captures a local (allocates a closure), string interpolation in a plain `Debug.Log` call (it still builds the
string when logging is off — `AppLogger` calls are removed entirely without `ENABLE_LOGS`), and
`GetComponents<T>()` returning a fresh array.

**Editor performance is bad, build is fine (or vice versa).**
The Editor adds overhead everywhere and holds extra copies of assets. Always confirm a problem in a
development build on the target device before optimizing it.

**Optimization made no measurable difference.**
You optimized something that wasn't the bottleneck, or you're GPU-bound and optimized CPU work.
Check which you are first — Unity 6.3's Highlights module details pane splits CPU and GPU directly.

**Frame rate drops the longer the game runs.**
Something is accumulating: a growing static collection, event subscribers never removed, pooled
objects never returned, or a leak of native memory. See
[Assets & Memory](UnityAssetsAndMemoryInstructions.md#troubleshooting).

---

# Summary: Quick Reference

## Always Do ✅
- Cache component references in Awake()
- Pre-allocate collections with expected capacity
- Use object pooling for frequently spawned objects
- Use non-allocating physics methods
- Cache shader property IDs
- Use ProfilerMarker for performance-critical code
- Use UniTask over coroutines for async gameplay code
- Destroy the Textures, Materials and Meshes you create at runtime
- Remove empty Unity lifecycle methods

## Never Do ❌
- GetComponent/Find in Update loops
- Allocate (new) in Update loops
- Use LINQ in Update loops
- String concatenation in hot paths
- Access Camera.main every frame
- Create WaitForSeconds in coroutine loops
- Use .material when .sharedMaterial suffices

## Consider ⚠️
- Throttle expensive operations
- Batch physics queries
- Profile on target platform
- Use Jobs/Burst for heavy computation
- Configure physics collision matrix
- Tick scope-lifetime C# logic with VContainer `ITickable`, runtime-lifetime work with R3 `EveryUpdate`
- Enable the GPU Resident Drawer for large static scenes
- Set `Application.targetFrameRate` explicitly (and vSync to 0)
- Suspend GC across a known-bounded no-hitch section

---

# Version Information

- **Target Unity Version**: Unity 6.3 (6000.3.x) and later
- **C# Version**: C# 9.0+ features supported
- **Last Updated**: September 2026

---

# Learn more

- [Optimizing your game performance](https://docs.unity3d.com/6000.3/Documentation/Manual/performance-profiling-tools.html)
- [Profiler window](https://docs.unity3d.com/6000.3/Documentation/Manual/Profiler.html)
- [Understanding optimization in Unity](https://unity.com/how-to/best-practices-performance-optimization-unity)
- [Unity Performance Tuning Bible](https://cyberagentgameentertainment.github.io/UnityPerformanceTuningBible/en/) — CyberAgent Game Entertainment
