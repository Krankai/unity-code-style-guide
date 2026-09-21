# Unity Scenes, Bootstrapping & Application Lifecycle

> **General Unity best practice.** Applies to any Unity 6 project. Change these only if you
> know why. Personal style preferences live in [`UnityStyleGuide.md`](../UnityStyleGuide.md);
> project-specific settings live in [`UnityCustomInstructions/`](../UnityCustomInstructions/).

How the game starts, how scenes come and go, and what survives in between.

Two problems dominate this area. The first is **initialization order** — something needs a service
that doesn't exist yet. The second is **state that outlives what created it** — a static field, an
event subscription, or a `DontDestroyOnLoad` object that's still there on the second playthrough.

Table of contents:
- [The bootstrap scene pattern](#the-bootstrap-scene-pattern)
- [Additive scene loading](#additive-scene-loading)
- [Initialization order](#initialization-order)
- [Domain reload and static state](#domain-reload-and-static-state)
- [DontDestroyOnLoad discipline](#dontdestroyonload-discipline)
- [Cross-scene references](#cross-scene-references)
- [Teardown](#teardown)
- [Troubleshooting](#troubleshooting)
- [Review checklist](#review-checklist)
- [Learn more](#learn-more)

---

## The bootstrap scene pattern

- ✅ Have one small scene that is always loaded first and never unloaded. It hosts the boot `LifetimeScope`, which
  registers the services that must exist for the whole session: audio, save system, input, scene loading,
  analytics. See
  [Dependency injection: VContainer](UnityArchitectureInstructions.md#dependency-injection-vcontainer).
- ✅ Everything else loads **additively** on top of it. Because the boot scene is never unloaded, its scope needs no
  `DontDestroyOnLoad`.
- ✅ Put it at index 0 in Build Settings so a build always starts there.
- ❌ After boot, never load a scene with `LoadSceneMode.Single`. It unloads the boot scene, and the services with it.
- ✅ After each additive load, call `SetActiveScene` on the new scene (see
  [Additive scene loading](#additive-scene-loading)).

### Parent link for the next scope

Each scene has its own `LifetimeScope`. To resolve the boot scope's services, the boot scope has to be its
**parent**. A scope takes its parent when it is created, and that happens when its scene *activates* (its `Awake`).

| Mechanism | How | Use when |
|---|---|---|
| `LifetimeScope.EnqueueParent(bootScope)` at the load call (this project's default) | A `using` block around the call that activates the scene — see the two variants under [Additive scene loading](#additive-scene-loading) | You control the code that loads the scene |
| **Parent** set to the boot scope's type in the scene scope's Inspector | VContainer looks the parent up among the loaded scenes when the scope is created | The scene may be loaded by code you don't control. The boot scope must already be built, or VContainer throws `VContainerParentTypeReferenceNotFound` |

- ⚠️ A Parent type set in the Inspector **wins over** `EnqueueParent`. Use one mechanism per scope, not both.
- ⚠️ `EnqueueParent` parents *any* scope created while its block is open, so keep the block as short as the
  activation it covers.
- ℹ️ The boot scope registers itself, so a service registered in it can take `LifetimeScope` as a constructor
  dependency instead of a serialized field.

### Play always starts from the boot scene

- ✅ Set `EditorSceneManager.playModeStartScene` to the boot scene. Pressing Play then always starts there,
  whichever scene is open, so the boot scope exists before any other scope is built.
- ✅ Do it from an Editor-only script that re-applies it on every domain reload:

```csharp
// Editor/PlayFromBootScene.cs
[InitializeOnLoad]
public static class PlayFromBootScene
{
    private const string BootScenePath = "Assets/Scenes/BootScene.unity";

    static PlayFromBootScene()
    {
        EditorSceneManager.playModeStartScene = AssetDatabase.LoadAssetAtPath<SceneAsset>(BootScenePath);
    }
}
```

- ℹ️ To run one scene in isolation, set `EditorSceneManager.playModeStartScene = null`; it is applied again on the
  next domain reload.
- ⚠️ Don't load the boot scene additively behind the open scene from a
  `[RuntimeInitializeOnLoadMethod(BeforeSceneLoad)]` guard. The open scene's scope would be built without a wrapped
  load to take its parent from, and the boot scene's own `Awake` isn't guaranteed to run first.

---

## Additive scene loading

- ✅ Load scenes additively and unload the one you leave. `Addressables.LoadSceneAsync(..., LoadSceneMode.Additive)`
  and `Addressables.UnloadSceneAsync` are this project's preferred way; `SceneManager.LoadSceneAsync` /
  `UnloadSceneAsync` for a scene that is only in Build Settings.
- ✅ Reference the scene through a `SceneReference` field (`.Address` for an Addressable scene) rather than a name
  string, when Eflatun.SceneReference is installed — see [UnityTechStack.md](../UnityCustomInstructions/UnityTechStack.md).
- ✅ Split large levels into additive chunks — geometry, lighting, gameplay, UI — so they can load and unload
  independently.
- ⚠️ `LoadSceneMode.Single` destroys everything not marked `DontDestroyOnLoad`, including the boot scene and its
  `LifetimeScope`. After boot, always `Additive`.
- ✅ Load in the background and activate at a moment you choose — this is how you avoid a hitch mid-gameplay.
  Addressables: `activateOnLoad: false`, then `SceneInstance.ActivateAsync()`. `SceneManager`:
  `allowSceneActivation = false`.
- ⚠️ `activateOnLoad: false` blocks the AsyncOperation queue, **including other Addressable loads**, until the scene
  is activated (Unity's docs). Keep the wait between "loaded" and "activate" short.
- ℹ️ Progress: an Addressables handle's `PercentComplete` is already 0–1, and the handle completes once the scene
  is loaded but not yet active. `SceneManager` progress stalls at `0.9` while `allowSceneActivation` is false — that
  is expected, not a bug; treat `>= 0.9` as "ready".
- ✅ Call `Resources.UnloadUnusedAssets()` after unloading, while the loading screen is still up.
- ✅ Where the `EnqueueParent` block goes depends on `activateOnLoad`, because the scene's scope is created when the
  scene *activates*:
  - `false`: wrap only the `ActivateAsync()` call — that is when the scope is created.
  - `true` (the default): wrap the **whole** `LoadSceneAsync` call, since activation happens whenever the load
    finishes.

```csharp
[SerializeField] private LifetimeScope _bootScope;   // parent for the level's own scope
private AsyncOperationHandle<SceneInstance> _levelHandle;

// activateOnLoad: false - background load, activate at a moment we choose
public async UniTask TaskLoadLevel(string address, CancellationToken token)
{
    try
    {
        _levelHandle = Addressables.LoadSceneAsync(address, LoadSceneMode.Additive, activateOnLoad: false);

        // The handle completes once the scene is loaded but still inactive
        while (!_levelHandle.IsDone)
        {
            _progressBar.value = _levelHandle.PercentComplete;   // already 0..1, no "divide by 0.9"
            await UniTask.NextFrame(token);
        }

        if (_levelHandle.Status != AsyncOperationStatus.Succeeded)
        {
            AppLogger.LogError($"{DebugPrefix} Failed to load scene '{address}'.", this);
            return;
        }

        await TaskFadeOut(token);

        // The level's LifetimeScope is created during activation, so only this call needs the block
        using (LifetimeScope.EnqueueParent(_bootScope))
        {
            await _levelHandle.Result.ActivateAsync();
        }

        SceneManager.SetActiveScene(_levelHandle.Result.Scene);
        await Resources.UnloadUnusedAssets();   // after the previous level was unloaded
        await TaskFadeIn(token);
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

```csharp
// activateOnLoad: true (the default) - the scene activates as soon as it has loaded, at a moment we
// don't control, so the whole load call sits inside the EnqueueParent block
public async UniTask TaskLoadLevelImmediately(string address)
{
    using (LifetimeScope.EnqueueParent(_bootScope))
    {
        await Addressables.LoadSceneAsync(address, LoadSceneMode.Additive);
    }
}
```

```csharp
// Build Settings scene through SceneManager: the same idea, with allowSceneActivation
AsyncOperation load = SceneManager.LoadSceneAsync(sceneName, LoadSceneMode.Additive);
load.allowSceneActivation = false;

// 0.9 is "loaded but not activated" - it will not reach 1.0 until we allow activation
while (load.progress < 0.9f)
{
    _progressBar.value = load.progress / 0.9f;
    await UniTask.NextFrame(token);
}

await TaskFadeOut(token);

using (LifetimeScope.EnqueueParent(_bootScope))
{
    load.allowSceneActivation = true;
    await load;
}
```

- ✅ Await the `AsyncOperationHandle` directly, not `.ToUniTask()` — UniTask documents a `Start()`-ordering caveat for
  the latter.
- ℹ️ `EnqueueParent` around `Addressables.LoadSceneAsync` is treated as working for now; it has not been confirmed
  by a project test yet.
- ℹ️ `SetActiveScene` matters: newly instantiated objects and lightmap/skybox settings come from the
  active scene. Forgetting it is why a level sometimes loads with the previous scene's lighting.

---

## Initialization order

Unity's guarantees, in order, for a single scene load:

1. All `Awake()` on all objects
2. All `OnEnable()` on all objects
3. All `Start()` before the first frame

- ✅ **`Awake` for self, `Start` for others.** Cache your own components in `Awake`; touch other
  objects in `Start`. That ordering is guaranteed; anything finer is not.
- ⚠️ Order *within* a phase is undefined. Two `Awake` methods have no guaranteed relative order.
- ⚠️ Script Execution Order (Project Settings → Script Execution Order) works, but it's a global,
  invisible dependency. Use it sparingly and comment why.
- ✅ Prefer explicit initialization for anything order-sensitive. With VContainer a service's constructor
  dependencies exist before it does, so let the container order them; for work that must run at start-up,
  register an entry point (e.g. `IStartable`) rather than relying on lifecycle timing.
- ⚠️ **Objects loaded additively run their `Awake` when that scene finishes loading**, not when the
  first scene did. A service registered in an additive scene isn't available to the base scene's
  `Start`. App-lifetime services live in the boot scope, which is built before any later scene loads, so this
  only bites a service registered in a level scene's own scope.

| Attribute | Runs |
|---|---|
| `[RuntimeInitializeOnLoadMethod(SubsystemRegistration)]` | Earliest — before subsystems register |
| `[RuntimeInitializeOnLoadMethod(BeforeSceneLoad)]` | Before the first scene's `Awake` |
| `[RuntimeInitializeOnLoadMethod(AfterSceneLoad)]` | After the first scene loads, before `Start` |
| `[InitializeOnLoad]` | Editor only, on domain reload |

---

## Domain reload and static state

Disabling Domain Reload (Project Settings → Editor → Enter Play Mode Options) saves 2–5 seconds on
every Play. The cost is that **static state no longer resets**.

- ⚠️ Static fields keep their values from the previous play session.
- ⚠️ Static events keep their subscribers, so handlers accumulate and fire multiple times.
- ⚠️ Singleton instances point at destroyed objects — the classic
  `MissingReferenceException` on second play.
- ✅ Reset every static explicitly. Make it a rule, not a case-by-case fix:

```csharp
public class GameSession
{
    private static int _score;
    private static List<Player> _players = new();

    public static event Action<int> ScoreChanged;

    // Runs on every play, whether or not domain reload is enabled
    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]
    private static void ResetStatics()
    {
        _score = 0;
        _players.Clear();
        ScoreChanged = null;   // Critical - otherwise last session's handlers survive
    }
}
```

- ✅ A static R3 `Subject` is reset the same way: dispose it and assign a new instance in `ResetStatics`.
- ✅ Keep the reset method physically next to the statics it resets. A reset method in a different
  file goes stale the moment someone adds a field.
- ✅ If a class has statics and no `ResetStatics`, treat that as a bug when domain reload is disabled.
- ℹ️ See [UnityProjectConfiguration.md](../UnityCustomInstructions/UnityProjectConfiguration.md#caveats-when-domain-reload-is-disabled)
  for the settings themselves.

---

## DontDestroyOnLoad discipline

- ⚠️ `DontDestroyOnLoad` on many objects is usually a sign of a missing bootstrap scene. A bootstrap
  scene that is never unloaded achieves the same thing, visibly, in one place.
- ⚠️ Objects moved to the `DontDestroyOnLoad` scene are easy to lose track of — they don't appear in
  any scene file, so nothing in version control records that they exist.
- ✅ If you do use it, apply it to a single root object and parent everything persistent under it.
- ⚠️ **Reloading the scene that created a `DontDestroyOnLoad` object creates a second one.** Guard it with a
  duplicate check in `Awake`, and reset the static in a `[RuntimeInitializeOnLoadMethod]` — the worked example is
  the `PaymentCallbackBridge` under
  [Singleton Pattern](UnityDesignPatternsInstructions.md#singleton-pattern).
- ℹ️ The boot scope is not one of these objects: it lives in the never-unloaded boot scene, so it needs no
  `DontDestroyOnLoad`. The `_hasConfigured` guard in the Architecture guide's `BootLifetimeScope` covers the one
  remaining risk, the boot scene being loaded a second time.

---

## Cross-scene references

- ❌ You cannot serialize a reference from an object in one scene to an object in another. Unity will
  either refuse the assignment or silently null it at runtime.
- ✅ Wire scenes together at runtime instead. Three options, in rough order of preference:

| Approach | Good for | Trade-off |
|---|---|---|
| A service injected through the boot scope (VContainer) | Long-lived services, anything one scene needs from another | The scene's scope must be parented to the boot scope — see [Parent link for the next scope](#parent-link-for-the-next-scope) |
| MessagePipe | The four reserved cases in the Architecture guide | Optional package; not a default channel |
| Objects register themselves on `OnEnable` with a service | Loose collections (enemies, spawn points) | Order-dependent if you read too early |

- ✅ Data that several scenes only *read* (tuning values, item definitions) belongs in a ScriptableObject `*Config`
  asset: both scenes reference the asset, neither references the other.
- ⚠️ Don't use a ScriptableObject to carry runtime state or events between scenes. Its runtime state persists in the
  Editor between plays — see [ScriptableObjects](UnityScriptableObjectInstructions.md) for config vs runtime data.

---

## Teardown

- ✅ Unsubscribe plain C# events in `OnDisable`; R3 `.AddTo(this)` subscriptions end on destroy. Dispose owned
  `Subject`s and release resources in `OnDestroy`. See [OnDestroy()](../UnityStyleGuide.md#ondestroy).
- ⚠️ Destruction order between objects is **not** guaranteed. Never assume another object is still
  alive in your `OnDestroy`.
- ⚠️ `OnDestroy` is not called if the object was never enabled.
- ⚠️ `OnApplicationQuit` does not run on mobile when the OS kills a backgrounded app. Save on
  `OnApplicationPause(true)` instead — that's the last callback you're reliably given.
- ✅ Use `Application.quitting` for static cleanup that has no MonoBehaviour to hang off.

```csharp
private void OnApplicationPause(bool isPaused)
{
    // On mobile this is the last reliable chance to persist state
    if (isPaused)
    {
        _saveService.SaveNow();
    }
}

private void OnApplicationQuit()
{
    // Desktop, and graceful mobile exits only
    _saveService.SaveNow();
}
```

---

## Troubleshooting

**Pressing Play on a gameplay scene throws null reference errors on services.**
The boot scene wasn't loaded first, so no boot scope exists. Set `playModeStartScene` as in
[Play always starts from the boot scene](#play-always-starts-from-the-boot-scene).

**A scene's scope can't resolve app-lifetime services, or throws `VContainerParentTypeReferenceNotFound`.**
The scope has no parent. Check that `EnqueueParent` wraps the call that *activates* the scene (`ActivateAsync()`
with `activateOnLoad: false`, the whole load call with `true`), and that no Parent type is set in the Inspector
unless the boot scope is already built — see [Parent link for the next scope](#parent-link-for-the-next-scope).

**Every service is gone after loading a scene.**
The scene was loaded with `LoadSceneMode.Single`, which unloaded the boot scene. After boot, load additively.

**Works the first time you press Play, breaks the second time.**
Classic disabled-Domain-Reload symptom. A static field or static event kept its value from the
previous session. Add a `ResetStatics` method — see
[Domain reload and static state](#domain-reload-and-static-state).

**An event handler fires two or three times, increasing each play session.**
Static event subscribers accumulated. Setting the event to `null` in `ResetStatics` is the fix; the
subscription itself is probably fine.

**Loading bar reaches 90% and stops.**
Expected with `SceneManager`. With `allowSceneActivation = false`, progress caps at `0.9`. Treat `>= 0.9` as ready and
divide by `0.9` for the display value.

**Objects instantiate into the wrong scene, or lighting is wrong after an additive load.**
`SetActiveScene` wasn't called. New objects and lighting settings come from the active scene.

**Two copies of a manager exist after reloading a scene.**
A `DontDestroyOnLoad` object created by a scene that got reloaded. Add the duplicate guard in
`Awake`.

**`MissingReferenceException` during scene teardown.**
`OnDestroy` touched another object that was already destroyed. Destruction order is not guaranteed —
don't reach across objects in `OnDestroy`.

**Save works on desktop, loses data on mobile.**
Saving only in `OnApplicationQuit`, which the OS doesn't call when it kills a backgrounded app. Save
in `OnApplicationPause(true)`.
---

## Review checklist

| Check | Look for |
|---|---|
| Bootstrap | No entry scene; services created ad hoc in gameplay scenes |
| Editor entry | Play doesn't start from the boot scene (no `playModeStartScene` helper) |
| Parent link | A scene scope with no `EnqueueParent` and no Parent type, or `EnqueueParent` around the wrong call for its `activateOnLoad` |
| Statics | Static fields or events with no `ResetStatics` while domain reload is off |
| Static events | `ScoreChanged = null` missing from the reset |
| DontDestroyOnLoad | Used on many objects; no duplicate guard |
| Scene loading | `LoadSceneMode.Single` used after boot — it unloads the boot scene and its services |
| Scene loading | `activateOnLoad: false` left waiting while other Addressable loads need to run |
| Progress | `SceneManager` loading bar that never passes 90% (`allowSceneActivation` misunderstanding) |
| Active scene | `SetActiveScene` not called after an additive load |
| Mobile save | Saving only in `OnApplicationQuit` |

---

## Learn more

- [Scene management](https://docs.unity3d.com/6000.3/Documentation/Manual/scenes-working-with.html)
- [Addressables: load a scene](https://docs.unity3d.com/Packages/com.unity.addressables@2.3/manual/LoadingScenes.html)
- [Domain reloading](https://docs.unity3d.com/6000.3/Documentation/Manual/domain-reloading.html)
- [Order of execution for event functions](https://docs.unity3d.com/6000.3/Documentation/Manual/execution-order.html)
