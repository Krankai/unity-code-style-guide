# Architecture Patterns

> **Applies once a project has adopted VContainer** — this project's default for dependency
> injection, per [`UnityTechStack.md`](../UnityCustomInstructions/UnityTechStack.md). MessagePipe and
> R3 are separate, narrower add-ons within the same stack; each is called out below as optional where
> it applies. Personal style preferences live in [`UnityStyleGuide.md`](../UnityStyleGuide.md).

How this project structures a gameplay feature: the Model/View/Controller split, which layers need an
interface and which don't, how services are wired together with VContainer, why Singletons are
discouraged, and when to reach for MessagePipe or R3 instead of a direct reference.

Table of contents:
- [MVC layering](#mvc-layering)
- [Interfaces per layer](#interfaces-per-layer)
- [Services](#services)
- [Dependency injection: VContainer](#dependency-injection-vcontainer)
- [Singleton policy](#singleton-policy)
- [Communication: direct references by default](#communication-direct-references-by-default)
- [MessagePipe: the reserved cases](#messagepipe-the-reserved-cases)
- [Reactive callbacks: R3](#reactive-callbacks-r3)
- [Review checklist](#review-checklist)
- [Learn more](#learn-more)

---

## MVC layering

Every feature is split into three layers:

- **Model** — data only, scoped to a single entity/instance. Pure C# class or `struct`. No Unity
  lifecycle, no `MonoBehaviour`.
- **View** — visual/presentation only. Unity-native `MonoBehaviour`. Reads state to display it,
  forwards raw input, contains no gameplay rules.
- **Controller** — runtime logic. **Pure C# class** (or `static class` for stateless logic), never a
  `MonoBehaviour`. Owns the rules that decide *what happens*, orchestrating the Model and the View.

**Feature-wide configuration is a separate concept from the Model**, and stays a ScriptableObject: the
Model is per-entity runtime data (pure C#), while the ScriptableObject is the authored/serialized
settings for the feature as a whole — see
[UnityScriptableObjectInstructions.md](UnityScriptableObjectInstructions.md) for the naming and
`[CreateAssetMenu]` rules that apply to it. For a Leaderboard feature: `LeaderboardConfig` (max
entries, refresh interval) is a ScriptableObject; each `LeaderboardEntry` (player name, score, rank) is
a pure C# `struct` — that's the Model.

> ℹ️ **This repo also has an older MVP example**, in
> [UnityUIToolkitInstructions.md](UnityUIToolkitInstructions.md#mvp-design-pattern-with-data-binding),
> where the Presenter is a `MonoBehaviour` and the Model is a ScriptableObject holding runtime-bound
> state. That example predates this guide and is kept as-is, as an illustration of UI Toolkit data
> binding — not as this project's default architecture. The MVC split above, with a pure C# Controller,
> is the current default.

```csharp
// Feature-wide config - ScriptableObject, authored/serialized
[CreateAssetMenu(fileName = "LeaderboardConfig", menuName = "Leaderboard/Leaderboard Config")]
public class LeaderboardConfig : ScriptableObject
{
    [SerializeField] private int _maxEntries;
    [SerializeField] private float _refreshIntervalInSeconds;

    public int MaxEntries => _maxEntries;
    public float RefreshIntervalInSeconds => _refreshIntervalInSeconds;
}

// Model - pure C# struct, one per entry
public struct LeaderboardEntry
{
    public string PlayerName;
    public int Score;
    public int Rank;
}

// View - MonoBehaviour, display only, behind an interface (see Interfaces per layer below)
public interface ILeaderboardView
{
    void Render(IReadOnlyList<LeaderboardEntry> entries);
}

public class LeaderboardView : MonoBehaviour, ILeaderboardView
{
    [SerializeField] private Transform _entryContainer;
    [SerializeField] private LeaderboardEntryView _entryPrefab;

    public void Render(IReadOnlyList<LeaderboardEntry> entries)
    {
        // instantiate/update _entryPrefab per entry
    }
}

// Controller - pure C#, owns the logic, depends on the View through its interface
public interface ILeaderboardController
{
    void Refresh();
}

public class LeaderboardController : ILeaderboardController
{
    private readonly ILeaderboardView _view;
    private readonly LeaderboardConfig _config;
    private List<LeaderboardEntry> _entries = new();

    public LeaderboardController(ILeaderboardView view, LeaderboardConfig config)
    {
        _view = view;
        _config = config;
    }

    public void Refresh()
    {
        // fetch/sort up to _config.MaxEntries entries into _entries
        _view.Render(_entries);
    }
}
```

---

## Interfaces per layer

- ✅ **Controller: interface always required.** Every Controller is defined by an interface, with a
  concrete implementation behind it — consumers depend on the interface, never the concrete type.
- ✅ **View: interface required where feasible.** A View's `MonoBehaviour` implements an interface
  (e.g. `LeaderboardView : MonoBehaviour, ILeaderboardView`), and Controllers depend on the interface,
  not the concrete `MonoBehaviour` — this keeps Controllers testable with a fake View. The
  `[SerializeField]` reference to the concrete `MonoBehaviour` still has to exist somewhere (a scene
  installer/`LifetimeScope` wiring it up), but that concrete reference stays at the wiring boundary,
  not inside the Controller.
- ✅ **Model: interface preferred, but not absolute.** Use an interface unless the data is genuinely
  lightweight, generic, and unlikely to change even long-term (e.g. a simple `Vector2Int GridPosition`
  struct) — in that case a plain `struct`/class is fine.

---

## Services

Each feature/system is a service, defined by its own interface — e.g. `IAnalyticService`,
`IInventoryService`, `IPurchaseService`. A service:
- Exposes its capability through the interface only.
- Is implemented by exactly one concrete class per environment (production vs. a fake/editor
  implementation is a valid reason for more than one).
- Is registered with and resolved through the DI container — never `new`'d directly by its consumers.

```csharp
public interface IAnalyticService
{
    void LogEvent(string eventName);
}

public class AnalyticService : IAnalyticService
{
    public void LogEvent(string eventName)
    {
        // send to analytics backend
    }
}
```

---

## Dependency injection: VContainer

Services are wired together through [VContainer](https://vcontainer.hadashikick.jp/), not constructed
manually or accessed through statics. This project uses a **persistent boot scope**: one root
`LifetimeScope` that registers every app-lifetime singleton and outlives every scene, with each later
scene's `LifetimeScope` parented to it.

- ✅ **`BootScene` is the first scene the app ever loads.** Its `LifetimeScope` registers every
  app-lifetime singleton service (`IAudioService`, `IAnalyticService`, and so on) and calls
  `DontDestroyOnLoad(gameObject)` on itself in `Configure`, before anything else runs.
- ✅ BootScene then loads the next scene with `LoadSceneMode.Single`. BootScene's own scene unloads,
  but the root `LifetimeScope` GameObject already moved out via `DontDestroyOnLoad`, so it survives.
- ✅ Every scene loaded after BootScene becomes a **child** of the boot scope, via
  `LifetimeScope.EnqueueParent(...)` wrapped around the `SceneManager.LoadSceneAsync(...)` call that
  loads it. A child scope resolves anything it doesn't register itself from its parent, so any later
  scene can take `IAudioService` as a constructor dependency without re-registering it.
- ⚠️ **This depends on BootScene genuinely being the first scene loaded, every time** — nothing in the
  pattern itself enforces that. If something ever reloads BootScene mid-session (a "return to title"
  flow gone wrong, for instance), guard `Configure` against running twice, or every app-lifetime
  singleton gets registered a second time.
- ✅ Register an interface against its concrete implementation; consumers request the interface via
  constructor injection, the same as any other service.
- ℹ️ **If this project uses [Eflatun.SceneReference](https://github.com/starikcetin/Eflatun.SceneReference)**
  (optional — a `SceneReference` serialized field replacing scene names/build indices as magic
  strings), reference the scene to load through it rather than a string, and load it through
  whichever API matches how that `SceneReference` is set up — `Addressables.LoadSceneAsync(reference.Address, ...)`
  for a scene marked Addressable (this project's preferred way to reference scenes), or
  `SceneManager.LoadSceneAsync(reference.Path, ...)` for one that's only in Build Settings. The package
  itself doesn't provide a loading method — it only exposes the identifiers Unity's/Addressables'
  existing async APIs already take.

```csharp
public class BootLifetimeScope : LifetimeScope
{
    private static bool _hasConfigured;

    [SerializeField] private SceneReference _mainMenuScene;

    protected override void Configure(IContainerBuilder builder)
    {
        if (_hasConfigured)
        {
            throw new System.InvalidOperationException("BootLifetimeScope must only be configured once.");
        }
        _hasConfigured = true;

        DontDestroyOnLoad(gameObject);

        builder.Register<IAudioService, AudioService>(Lifetime.Singleton);
        builder.Register<IAnalyticService, AnalyticService>(Lifetime.Singleton);
    }

    private void Start()
    {
        TaskLoadMainMenu().Forget();
    }

    private async UniTaskVoid TaskLoadMainMenu()
    {
        using (LifetimeScope.EnqueueParent(this))
        {
            // _mainMenuScene.Address is only valid when the scene is marked Addressable in the
            // Inspector. Await the AsyncOperationHandle directly here, not .ToUniTask() - the same
            // Start()-ordering caveat UniTask documents for SceneManager.LoadSceneAsync applies to
            // Addressables.LoadSceneAsync too.
            await Addressables.LoadSceneAsync(_mainMenuScene.Address, LoadSceneMode.Single);
        }
    }
}

// Registers only what this scene needs - IAudioService/IAnalyticService resolve
// automatically from the boot parent, no re-registration required
public class MainMenuLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<IMainMenuController, MainMenuController>(Lifetime.Scoped);
    }
}
```

- ⚠️ This repo's **older** reference material shows a hand-rolled `ServiceLocator` (a static registry,
  resolved by calling `ServiceLocator.Resolve<T>()` from `Awake`) in
  [UnityDesignPatternsInstructions.md](UnityDesignPatternsInstructions.md), and a `ServiceLocator.Unregister(this)`
  call in `AGENTS.md`'s `OnDestroy` example. Both predate this guide. VContainer's constructor
  injection is the current default — a service is a constructor parameter, not something pulled from
  a static locator at runtime.
- ⚠️ **Unverified — check this before relying on it:** `EnqueueParent`'s documented behavior is that it
  parents *any* `LifetimeScope` instantiated while its `using` block is open, not something tied
  specifically to `SceneManager.LoadSceneAsync`. That should mean it works identically when the load is
  triggered by `Addressables.LoadSceneAsync` instead, since both ultimately instantiate the new scene's
  objects — including its `LifetimeScope` — through the same underlying Unity scene-load machinery. I
  could not find a primary-source VContainer example combining `EnqueueParent` with
  `Addressables.LoadSceneAsync` specifically to confirm this directly, so verify it behaves as expected
  the first time this project combines the two, rather than assuming it from this guide alone.

---

## Singleton policy

Singletons (`Instance` statics) are **discouraged**. If one is genuinely needed, it's not a silent
decision — the code introducing it must **specifically state its usage and the reason it was
necessary**, so the choice is visible and reviewable rather than just present.

- ✅ State the reason at the point of declaration, as a comment — not in a commit message or a design
  doc someone has to go find.
- ✅ Before reaching for a Singleton, check whether VContainer already solves the actual problem: a
  service registered `Lifetime.Singleton` in the boot scope (see
  [Dependency injection: VContainer](#dependency-injection-vcontainer)) gives the same "one instance,
  globally reachable" property, resolved through constructor injection instead of a static field. Most
  of what a Singleton is used for in ad-hoc code is exactly this — which is why a genuine case for one
  should be rare on this project, not the default reach.
- ℹ️ A Singleton might still be justified for something that isn't really a service-lifetime problem at
  all — e.g. a static bridge a legacy or third-party plugin's API requires by contract, where there's
  no constructor for VContainer to inject into in the first place.

---

## Communication: direct references by default

Default to **direct references / DI-resolved interfaces** for communication, including across
features — a Controller that needs another feature's service just takes that service's interface as a
constructor dependency, the same as any other service. For most projects on this stack, VContainer
alone covers the large majority of cross-system communication needs.

- ⚠️ This repo's **older** reference material treats a static `StaticGameEvents` bus (in
  [UnityDesignPatternsInstructions.md](UnityDesignPatternsInstructions.md)) as the default
  cross-system channel. That predates this guide. A direct, constructor-injected interface reference
  is the current default; `StaticGameEvents`-style broadcasting is what
  [MessagePipe](#messagepipe-the-reserved-cases) replaces, for the specific cases below — not a
  general-purpose default.

---

## MessagePipe: the reserved cases

**MessagePipe is optional** — install it only if/when one of the cases below actually comes up; a
project that never hits one of these never needs it. Please verify this list matches what you have in
mind:

- **Avoiding a circular/backward dependency** — a lower-level system needs to notify something that
  already depends on it, and a direct reference the other way would create a cycle.
- **One-to-many fan-out with an unknown/variable number of listeners** — an event where any number of
  unrelated systems might care (UI, Analytics, Achievements, SaveSystem all reacting to the same
  "PlayerDied" event), and the publisher shouldn't need to know or hold references to all of them.
- **Decoupled/mismatched lifetimes** — publisher and subscriber are created and destroyed
  independently of each other (e.g. a transient popup that subscribes to a global event and may or may
  not exist yet when the event fires).
- **Genuinely broadcast-style, ownerless events** — global state transitions (`GameStarted`,
  `GamePaused`, `LevelCompleted`) that aren't naturally "owned" by any single service, so there's no
  single interface that would be the "right" one to depend on directly.

If none of these apply, use a directly-injected interface reference instead of MessagePipe.

MessagePipe best practices, when one of the above does apply:
- Define one message type per event, as an immutable `struct` or `readonly record struct` — never a
  shared generic "message object" with a type tag. Strict typing per message, no generics-as-envelope.
- Inject `IPublisher<T>`/`ISubscriber<T>` for the specific message type via the constructor, the same
  as any other service dependency.
- Always dispose subscriptions — collect them in a `DisposableBag`/`DisposableBagBuilder` and dispose
  on teardown (e.g. `LifetimeScope` disposal or `OnDestroy`), rather than holding a single
  `IDisposable` per subscription by hand.
- Register message types through VContainer's MessagePipe integration
  (`builder.RegisterMessagePipe()`, `builder.RegisterMessageBroker<T>(options)`), not the built-in
  container — keeps message lifetime consistent with the rest of the DI graph.
- Default to keyless `IPublisher<T>`/`ISubscriber<T>`. Reach for the keyed variant
  (`IPublisher<TKey, TMessage>`) only when messages genuinely need to be routed by an ID/topic (e.g.
  per-player events in a multiplayer session).
- Use MessagePipe's diagnostics (`MessagePipeDiagnosticsInfo`) during development to catch
  subscription leaks early.

Register MessagePipe itself, and a broker per message type, in whichever `LifetimeScope` matches that
message's actual reach. `PlayerDiedMessage` is exactly the fan-out case this section describes — UI,
Analytics, Achievements and SaveSystem all react to it, and none of them own it — so it's registered
in the boot scope (see [Dependency injection: VContainer](#dependency-injection-vcontainer)) rather
than a single feature's scene scope, the same reasoning as any other app-lifetime service:

```csharp
public class BootLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        // ... DontDestroyOnLoad, IAudioService, IAnalyticService, etc. - see Dependency injection

        MessagePipeOptions options = builder.RegisterMessagePipe();

        // Wires MessagePipeDiagnosticsInfo up for this container
        builder.RegisterBuildCallback(c => GlobalMessagePipe.SetProvider(c.AsServiceProvider()));

        // One call per message type - Unity/IL2CPP doesn't support open-generic auto registration
        builder.RegisterMessageBroker<PlayerDiedMessage>(options);
    }
}
```

Any consumer, in any scope parented to the boot scope, then takes `IPublisher<PlayerDiedMessage>` or
`ISubscriber<PlayerDiedMessage>` as a constructor dependency, the same as any other service:

```csharp
public readonly struct PlayerDiedMessage
{
    public readonly string PlayerId;
    public PlayerDiedMessage(string playerId) => PlayerId = playerId;
}

public class PlayerHealthController
{
    private readonly IPublisher<PlayerDiedMessage> _publisher;

    public PlayerHealthController(IPublisher<PlayerDiedMessage> publisher)
    {
        _publisher = publisher;
    }

    private void OnHealthDepleted(string playerId) => _publisher.Publish(new PlayerDiedMessage(playerId));
}

public class AchievementController
{
    private readonly IDisposable _subscription;

    public AchievementController(ISubscriber<PlayerDiedMessage> subscriber)
    {
        var bag = DisposableBag.CreateBuilder();
        subscriber.Subscribe(OnPlayerDied).AddTo(bag);
        _subscription = bag.Build();
    }

    private void OnPlayerDied(PlayerDiedMessage message) { /* unlock achievement, etc. */ }

    public void Dispose() => _subscription.Dispose();
}
```

---

## Reactive callbacks: R3

Use [R3](https://github.com/Cysharp/R3) `Observable` in place of raw `Action`/`UnityEvent` callbacks,
primarily for **deterministic subscription lifecycle management**, not for a raw allocation win over
`Action` (R3's allocation advantage is measured against legacy UniRx, not against plain C# delegates —
a `Subscribe()` call still allocates; the real gain is that cleanup is automatic via `AddTo()` instead
of something to remember in `OnDestroy`).

- ✅ This doesn't replace this repo's general `event Action`/`Action<T>` guidance for code-only events
  — see [Events](../UnityStyleGuide.md#events). Reach for R3 specifically when a subscription's
  cleanup would otherwise need to be tracked by hand.
- ✅ Subscribe and immediately attach the subscription to the object's lifetime: `.AddTo(this)`
  (component-scoped) or a `CompositeDisposable`/`DisposableBag` for manually-scoped groups.
- ✅ Prefer `.AsObservable(this.destroyCancellationToken)` on a `UnityEvent` over manual `+=`/`-=` in
  `OnEnable`/`OnDisable`.
- ✅ Use R3's composition operators (`Where`, `Select`, `Throttle`, `DistinctUntilChanged`) instead of
  hand-rolled filtering/debouncing logic inside a callback body.

```csharp
public class JumpButtonView : MonoBehaviour
{
    [SerializeField] private Button _jumpButton;
    private readonly CompositeDisposable _disposables = new();

    private void Start()
    {
        _jumpButton.OnClickAsObservable()
            .Subscribe(_ => OnJumpRequested())
            .AddTo(_disposables);
    }

    private void OnJumpRequested() { /* delegate to Controller */ }

    private void OnDestroy() => _disposables.Dispose();
}
```

---

## Review checklist

| Check | Look for |
|---|---|
| Controller | A Controller with no interface, or a Controller that's a `MonoBehaviour` |
| Model | Per-entity runtime data modelled as a ScriptableObject instead of a plain struct/class |
| Config | Feature-wide settings modelled as a plain C# class instead of a ScriptableObject |
| DI | A service constructed with `new` outside a `LifetimeScope`, or resolved via a static locator |
| DI | A scene's `LifetimeScope` with no `EnqueueParent` to the boot scope, so it can't resolve app-lifetime services |
| DI | `BootLifetimeScope.Configure` with no guard against running twice |
| Singleton | An `Instance` static with no comment stating why it was necessary |
| Communication | A new global static event bus reached for by default, instead of a direct reference |
| MessagePipe | MessagePipe used for a case that isn't one of the four reserved ones |
| MessagePipe | A message type that's a shared generic envelope instead of one immutable struct per event |
| MessagePipe | A subscription not collected in a `DisposableBag`/disposed on teardown |
| R3 | A manual `+=`/`-=` subscription with no matching cleanup, where R3 would make it automatic |

---

## Learn more

- [VContainer](https://vcontainer.hadashikick.jp/) — documentation for the DI container this project uses.
- [MessagePipe, GitHub](https://github.com/Cysharp/MessagePipe) — the pub/sub library, for the reserved cases above.
- [R3, GitHub](https://github.com/Cysharp/R3) — the reactive extensions library.
- [Eflatun.SceneReference, GitHub](https://github.com/starikcetin/Eflatun.SceneReference) — optional
  typed scene references, used in [Dependency injection: VContainer](#dependency-injection-vcontainer)
  above if this project has adopted it.
- [UnityUniTaskInstructions.md](UnityUniTaskInstructions.md) — this project's async default; Controllers
  and services commonly combine UniTask with the patterns above.
