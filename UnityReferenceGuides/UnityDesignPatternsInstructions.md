# SOLID & Design Patterns for Unity

> **General Unity best practice.** Applies to any Unity 6 project. Change these only if you
> know why. Personal style preferences live in [`UnityStyleGuide.md`](../UnityStyleGuide.md);
> project-specific settings live in [`UnityCustomInstructions/`](../UnityCustomInstructions/).


This document outlines some recommended design patterns for Unity development, along with some example code snippets.
The work is ongoing and will be updated over time. I'm sure there are many more patterns and best practices that could be added.
It's inspired by the ebook "Level up your code with design patterns and SOLID" I co-authored and which you can find here: https://unity.com/resources/design-patterns-solid-ebook

Intent is to provide a quick reference guide for when you need a refresher.
It complements the [style guide](../UnityStyleGuide.md) by providing more detailed guidance on specific patterns and their usage in Unity projects.

> **Cross-references:** For C# code style and naming conventions, see [UnityStyleGuide.md](../UnityStyleGuide.md). For this project's architecture — MVC layering, VContainer dependency injection, the Singleton policy, MessagePipe, and R3 — see [UnityArchitectureInstructions.md](UnityArchitectureInstructions.md); where an older example in this guide differs, that guide wins. For UI Toolkit patterns including data binding and MVP, see [UnityUIToolkitInstructions.md](UnityUIToolkitInstructions.md).

Table of Contents
=================

- [SOLID Principles](#solid-principles)
    - [Single Responsibility Principle](#single-responsibility-principle)
    - [Open/Closed Principle](#openclosed-principle)
    - [Liskov Substitution Principle](#liskov-substitution-principle)
    - [Interface Segregation Principle](#interface-segregation-principle)
    - [Dependency Inversion Principle](#dependency-inversion-principle)
- [Design Patterns for Unity](#design-patterns-for-unity)
    - [Worked Example: A Tile-Based Strategy Game](#worked-example-a-tile-based-strategy-game)
    - [Reference Patterns](#reference-patterns)
- [Observer Pattern](#observer-pattern)
- [State Pattern](#state-pattern)
    - [Class-Based State Pattern](#class-based-state-pattern)
    - [Enum-Based State Pattern](#enum-based-state-pattern)
- [Template Method Pattern](#template-method-pattern)
- [Singleton Pattern](#singleton-pattern)
- [Dependency Injection / Service Locator](#dependency-injection--service-locator)
- [Composition over Inheritance](#composition-over-inheritance)
- [Object Pooling](#object-pooling)
- [Factory Pattern](#factory-pattern)
- [Command Pattern](#command-pattern)
- [Strategy Pattern](#strategy-pattern)
- [Additional Resources](#additional-resources)

## SOLID Principles
- S - Single Responsibility Principle
- O - Open/Closed Principle
- L - Liskov Substitution Principle
- I - Interface Segregation Principle
- D - Dependency Inversion Principle

### Single Responsibility Principle
- A class should have only one reason to change, meaning it should only have one job or responsibility
- Break down large classes into smaller, focused classes that handle specific tasks

> See also: [Composition over Inheritance](#composition-over-inheritance) for how SRP applies to a tile system.

```csharp
[RequireComponent(typeof(PlayerAudio), typeof(PlayerInput),
typeof(PlayerMovement))]

public class Player : MonoBehaviour
{
    [SerializeField] private PlayerAudio _playerAudio;
    [SerializeField] private PlayerInput _playerInput;
    [SerializeField] private PlayerMovement _playerMovement;

    private void Awake()
    {
        _playerAudio = GetComponent<PlayerAudio>();
        _playerInput = GetComponent<PlayerInput>();
        _playerMovement = GetComponent<PlayerMovement>();
    }
}

public class PlayerAudio : MonoBehaviour
{
    // Handles all audio-related logic
}

public class PlayerInput : MonoBehaviour
{
    // Handles all input-related logic
}

public class PlayerMovement : MonoBehaviour
{
    // Handles all movement-related logic
}
```

### Open/Closed Principle
- Classes must be open for extension but closed for modification
- A classic example of this is calculating the area of a shape
- Structure your classes so that you can create new behavior without modifying the original code

```csharp
public abstract class Shape
{
    public abstract float CalculateArea();
}

public class Rectangle : Shape
{
    public float Width;
    public float Height;

    public override float CalculateArea()
    {
        return Width * Height;
    }
}

public class Circle : Shape
{
    public float Radius;

    public override float CalculateArea()
    {
        return Radius * Radius * Mathf.PI;
    }
}

// New shapes can be added without modifying AreaCalculator
public class AreaCalculator
{
    public float GetArea(Shape shape)
    {
        return shape.CalculateArea();
    }
}
```

### Liskov Substitution Principle
- Subtypes must be substitutable for their base types without altering the correctness of the program
- If a method accepts a base class, any derived class should work without unexpected behavior
- Avoid overriding methods in ways that violate the base class's expected behavior

> See also: [Interfaces](../UnityStyleGuide.md#interfaces) in the style guide for guidance on when to use interfaces versus abstract base classes.

```csharp
// Good: Both derived classes honor the base contract
public abstract class Unit
{
    public abstract int CalculateDamage();
}

public class InfantryUnit : Unit
{
    private int _baseDamage = 10;

    public override int CalculateDamage()
    {
        // Returns a positive damage value as expected by consumers
        return _baseDamage;
    }
}

public class CavalryUnit : Unit
{
    private int _baseDamage = 15;
    private int _chargeBonus = 5;

    public override int CalculateDamage()
    {
        // Also returns a positive damage value — substitutable for Unit
        return _baseDamage + _chargeBonus;
    }
}

// Any code that works with Unit also works with InfantryUnit or CavalryUnit
public void ApplyDamageToTarget(Unit attacker, IDamageable target)
{
    int damage = attacker.CalculateDamage();
    target.ApplyDamage(damage);
}
```

```csharp
// Bad: Violates LSP — RangedUnit changes the expected behavior
public class RangedUnit : Unit
{
    public override int CalculateDamage()
    {
        // Returns -1 when out of ammo — callers don't expect negative values
        if (_ammo <= 0) return -1;
        return _baseDamage;
    }
}
```

### Interface Segregation Principle
- No class should be forced to implement interfaces it doesn't use
- Prefer small, focused interfaces over large monolithic ones
- Each interface should represent a single capability or role

> See also: [Interfaces](../UnityStyleGuide.md#interfaces) in the style guide for naming conventions (`I` prefix, PascalCase).

```csharp
// Good: Small, focused interfaces
public interface IDamageable
{
    void ApplyDamage(int amount);
}

public interface IHealable
{
    void Heal(int amount);
}

public interface IMovable
{
    void MoveTo(Vector3 position);
}

// A unit that can take damage and move, but cannot be healed
public class SkeletonUnit : MonoBehaviour, IDamageable, IMovable
{
    public void ApplyDamage(int amount) { /* ... */ }
    public void MoveTo(Vector3 position) { /* ... */ }
}

// A building that can take damage but cannot move or be healed
public class WallStructure : MonoBehaviour, IDamageable
{
    public void ApplyDamage(int amount) { /* ... */ }
}
```

```csharp
// Bad: Forces every implementer to handle capabilities it may not have
public interface IEntity
{
    void ApplyDamage(int amount);
    void Heal(int amount);
    void MoveTo(Vector3 position);
    void Attack(IEntity target);
}
```

### Dependency Inversion Principle
- High-level modules should not depend on low-level modules; both should depend on abstractions
- Depend on interfaces or abstract classes rather than concrete implementations

> [Dependency injection](#dependency-injection--service-locator) is how this project decouples high-level systems from concrete dependencies at runtime — see [Dependency injection: VContainer](UnityArchitectureInstructions.md#dependency-injection-vcontainer).

```csharp
// Good: High-level logic depends on an abstraction
public interface IAudioService
{
    void PlaySound(string clipName);
}

public class UnityAudioService : IAudioService
{
    public void PlaySound(string clipName)
    {
        // Unity-specific audio playback
    }
}

// The controller depends on the interface, not the concrete class. VContainer supplies it
// through the constructor - no static lookup, no Awake() resolve
public class CombatController
{
    private readonly IAudioService _audioService;

    public CombatController(IAudioService audioService)
    {
        _audioService = audioService;
    }

    public void HandleAttackLanded()
    {
        _audioService.PlaySound("SwordHit");
    }
}
```

## Design Patterns for Unity

- ✅ Choose patterns pragmatically. Apply them when they solve a real problem or improve maintainability, not just for the sake of using a
  pattern.

### Worked Example: A Tile-Based Strategy Game

These patterns come from the tile-based strategy game this guide was written around. Rows marked **legacy** or **discouraged** are superseded by [UnityArchitectureInstructions.md](UnityArchitectureInstructions.md) for new code; match the rest for consistency. The `StaticGameEvents` calls in the examples further down belong to the legacy event hub.

| Pattern | Location | Purpose |
|---------|----------|---------|
| [Observer Pattern](#observer-pattern) | `StaticGameEvents.cs` | Centralized static event bus — **legacy** as the cross-system default (MessagePipe replaces it) |
| [State Pattern (Enum)](#enum-based-state-pattern) | `UIRootController.cs` | UI state machine with enum + switch |
| [Template Method](#template-method-pattern) | `UIViewBase.cs` | Base class for all UI Toolkit views |
| [Singleton](#singleton-pattern) | `UIRootController.cs` | Global access to UI state controller — **discouraged**, see the Singleton policy |
| [Dependency Injection](#dependency-injection--service-locator) | `ServiceLocator.cs` / `DependencyInjector.cs` | Runtime dependency resolution — the hand-rolled locator is **legacy**; VContainer is the default |
| [Composition](#composition-over-inheritance) | `Tile` + `TileGarrison` etc. | Decomposing tile logic into focused components |
| ScriptableObject Data | Various `*Config` classes | Static configuration data |
| Data Binding | `[CreateProperty]` + `dataSource` | UI Toolkit automatic UI updates |

### Reference Patterns

These patterns are documented for reference and may be useful for future features.

| Pattern | Use Case |
|---------|----------|
| [Class-Based State](#class-based-state-pattern) | Complex AI or character controllers with per-state logic |
| [Object Pooling](#object-pooling) | Frequently spawned/despawned objects |
| [Factory Pattern](#factory-pattern) | Centralized object creation |
| [Command Pattern](#command-pattern) | Undo/redo, action history, input replay |
| [Strategy Pattern](#strategy-pattern) | Interchangeable behaviors at runtime |

---

## Observer Pattern

- ✅ Implement the Observer pattern with R3: the publisher owns a private `Subject<T>` and exposes it as a
  read-only `Observable<T>`; subscribers call `.Subscribe(...)` and attach the result to their own lifetime with
  `.AddTo(...)`. See [Reactive callbacks: R3](UnityArchitectureInstructions.md#reactive-callbacks-r3).
- ✅ Only the owner raises the event (`OnNext`). Exposing `Observable<T>` instead of the `Subject<T>` enforces this,
  which is what the static invoke methods below did by hand.
- ✅ Use plain `event Action` only where an assembly can't reference R3.
- ✅ Prefer `Subject`/`Observable` over `UnityEvent`; keep `UnityEvent` for callbacks exposed to the Inspector.
- ✅ Subscribe in `Start` with `.AddTo(this)`. For a handler that must stop while the component is disabled,
  subscribe in `OnEnable` into a `CompositeDisposable` and `Clear()` it in `OnDisable`. Never `.AddTo(this)` in
  `OnEnable` — re-enabling would add a duplicate.
- ⚠️ For notifications *between* systems, the default is a DI-resolved interface, with MessagePipe for the
  reserved cases — see [Communication](UnityArchitectureInstructions.md#communication-direct-references-by-default).
  A static event hub is what MessagePipe replaces.
- ❌ Avoid using events for tightly coupled systems where a direct method call is simpler.

> See also: [Events](../UnityStyleGuide.md#events) in the style guide for naming conventions and subscription patterns.

**Worked example — a tile announcing a population change:**

```csharp
// Publisher: owns the Subject, exposes only the Observable
public class Tile : MonoBehaviour
{
    private readonly Subject<int> _populationChanged = new();
    public Observable<int> OnPopulationChanged => _populationChanged;

    private int _population;

    public void ChangePopulation(int delta)
    {
        _population += delta;
        _populationChanged.OnNext(_population);
    }

    private void OnDestroy() => _populationChanged.Dispose();
}

// Subscriber: attaches to its own lifetime
public class TilePopulationLabel : MonoBehaviour
{
    [SerializeField] private Tile _tile;
    [SerializeField] private TMP_Text _label;

    private void Start()
    {
        _tile.OnPopulationChanged.Subscribe(HandlePopulationChanged).AddTo(this);
    }

    private void HandlePopulationChanged(int population)
    {
        _label.SetText("{0}", population);
    }
}
```

**Why this pattern:** the tile doesn't know the label exists, and the label's subscription cleans itself up when it
is destroyed, so there is no `OnDisable` to forget. Compared with `event Action`, the subscription is an
`IDisposable`, so it also composes with R3's operators (`Where`, `Throttle`, `DistinctUntilChanged`).

**Legacy worked example — a centralized static event hub (`StaticGameEvents.cs`):**

> ⚠️ This is the hub the Architecture guide's [MessagePipe](UnityArchitectureInstructions.md#messagepipe-the-reserved-cases) section replaces. It is kept for reading older code — don't add new cross-system events to it by default. It is also plain `event Action`, and its `OnTurnEnded`-style event names predate the current naming rules (an `event Action` is past tense without `On`; `On` + past tense names an R3 observable). Static events hold their subscribers for the life of the app, so a missed unsubscribe leaks (see [Troubleshooting](#troubleshooting)).

```csharp
// StaticGameEvents.cs — Centralized event bus (actual project pattern)
public static class StaticGameEvents
{
    // Events are public for subscribing, but invocation is controlled via static methods
    public static event Action<Tile> OnTileSelected;
    public static event Action<ArmyController> OnArmySelected;
    public static event Action OnTurnStarted;
    public static event Action OnTurnEnded;
    public static event Action<UIScreen> OnUIStateChanged;
    public static event Action OnResourcesChanged;

    // Static methods control invocation — external code cannot fire events directly
    public static void InvokeOnTileSelected(Tile tile) => OnTileSelected?.Invoke(tile);
    public static void InvokeOnTurnEnded() => OnTurnEnded?.Invoke();
    public static void InvokeOnUIStateChanged(UIScreen newState) => OnUIStateChanged?.Invoke(newState);
    public static void InvokeOnResourcesChanged() => OnResourcesChanged?.Invoke();
}
```

**Subscribing from a consumer:**

```csharp
public class Tile : MonoBehaviour
{
    private void OnEnable()
    {
        StaticGameEvents.OnTurnEnded += CalculateEndOfTurnDif;
    }

    private void OnDisable()
    {
        StaticGameEvents.OnTurnEnded -= CalculateEndOfTurnDif;
    }

    public void CalculateEndOfTurnDif()
    {
        CalculateDeltaEffectPerTurn();
        ApplyEndOfTurnDiff();
    }
}
```

**Why this pattern:** A single static event class avoids scattered event declarations across multiple managers. The static invoke methods ensure events are only raised by authorized code paths, not by arbitrary subscribers. The trade-off is global mutable state with no owner, which is why the Architecture guide moves broadcast-style events to MessagePipe's typed brokers instead.

---

## State Pattern

Use the State pattern for complex state-dependent behavior, such as character controllers, AI, or UI flows.

> See also: [Use enums for managing states](../UnityStyleGuide.md#use-enums-for-managing-states) in the style guide for enum naming conventions.

### Class-Based State Pattern

- ✅ Use when each state has substantial, distinct logic (e.g., AI with complex per-state Update/Enter/Exit behavior).
- ✅ Define an abstract base state class with `Enter()`, `Update()`, and `Exit()` methods.

```csharp
// Base state class
public abstract class PlayerState
{
    protected PlayerController _controller;

    public PlayerState(PlayerController controller)
    {
        _controller = controller;
    }

    public abstract void Enter();
    public abstract void Update();
    public abstract void Exit();
}

// Concrete state
public class IdleState : PlayerState
{
    public IdleState(PlayerController controller) : base(controller) { }

    public override void Enter() { /* Start idle animation */ }
    public override void Update() { /* Check for input to transition */ }
    public override void Exit() { /* Clean up idle state */ }
}

// Controller manages state transitions
public class PlayerController : MonoBehaviour
{
    private PlayerState _currentState;
    private IdleState _idleState;
    private RunningState _runningState;

    private void Awake()
    {
        _idleState = new IdleState(this);
        _runningState = new RunningState(this);
        _currentState = _idleState;
    }

    private void Update()
    {
        _currentState.Update();
    }

    public void ChangeState(PlayerState newState)
    {
        _currentState.Exit();
        _currentState = newState;
        _currentState.Enter();
    }
}
```

### Enum-Based State Pattern

- ✅ Prefer enums + switch when states primarily control which panels/behaviors are active rather than having complex per-state logic.
- ✅ This is the approach used by `UIRootController` in the worked example below.

```csharp
// Enum-based state pattern (actual project pattern from UIRootController.cs)
public enum UIScreen
{
    DefaultMapView,
    ArmyView,
    ArmyRecruitmentView,
    EventPopupView,
    DetailView,
    TownConstructionView,
    ConquestView
}

public class UIRootController : MonoBehaviour
{
    [SerializeField] private UIScreen _currentState = UIScreen.DefaultMapView;

    public UIScreen CurrentState => _currentState;

    public void ChangeState(UIScreen newState)
    {
        _currentState = newState;
        StaticGameEvents.InvokeOnUIStateChanged(CurrentState);
        ApplyStateToPanels(newState);
    }

    private void ApplyStateToPanels(UIScreen state)
    {
        // Hide all panels first, then enable the ones for the current state
        HideAllPanels();

        switch (state)
        {
            case UIScreen.DefaultMapView:
                SetPanelsActive(_resourceControllerView, true);
                SetPanelsActive(_gameTurnControllerView, true);
                SetPanelsActive(_logPanelView, true);
                break;

            case UIScreen.ArmyView:
                SetPanelsActive(_resourceControllerView, true);
                SetPanelsActive(_commanderView, true);
                SetPanelsActive(_armyLowerPanelView, true);
                break;

            // Additional states follow the same pattern...
        }
    }
}
```

**When to choose which approach:**

| Criteria | Enum + Switch | Class-Based |
|----------|--------------|-------------|
| State count | Few (< 10) | Many or growing |
| Per-state logic | Minimal (toggle panels) | Complex (different Update loops) |
| Transitions | Simple, centralized | Complex, conditional |
| Example use case | UI view management | AI behavior, character controllers |

---

## Template Method Pattern

- ✅ Use the Template Method pattern to define a skeleton algorithm in a base class, letting subclasses fill in the specific steps.
- ✅ This ensures consistent lifecycle management across all subclasses while allowing each to customize behavior.

**Worked example — `UIViewBase` as the base for all UI Toolkit views:**

```csharp
// UIViewBase.cs — Base class for all UI Toolkit panels (actual project pattern)
public abstract class UIViewBase : MonoBehaviour
{
    protected UIDocument _uiDocument;
    protected VisualElement _rootVisualElement;

    protected virtual void Awake()
    {
        _uiDocument = GetComponent<UIDocument>();
        _rootVisualElement = _uiDocument.rootVisualElement;
        InitializeElements();   // Step 1: subclass caches UI elements
    }

    protected virtual void OnEnable()
    {
        RegisterCallbacks();    // Step 2: subclass subscribes to events
    }

    protected virtual void OnDisable()
    {
        UnregisterCallbacks();  // Step 3: subclass cleans up subscriptions
    }

    // Abstract steps that each UI view must implement
    protected abstract void InitializeElements();
    protected abstract void RegisterCallbacks();
    protected abstract void UnregisterCallbacks();
    public abstract void ShowPanel(bool show);
}
```

**Creating a new UI view inheriting from the base class:**

```csharp
// Example: A concrete UI panel following the template
public class TileView : UIViewBase
{
    private VisualElement _tilePanel;
    private Label _populationLabel;

    protected override void InitializeElements()
    {
        _tilePanel = _rootVisualElement.Q<VisualElement>("tile-panel");
        _populationLabel = _rootVisualElement.Q<Label>("population-label");
    }

    protected override void RegisterCallbacks()
    {
        StaticGameEvents.OnTileSelected += HandleTileSelected;
    }

    protected override void UnregisterCallbacks()
    {
        StaticGameEvents.OnTileSelected -= HandleTileSelected;
    }

    public override void ShowPanel(bool show)
    {
        _tilePanel.style.display = show ? DisplayStyle.Flex : DisplayStyle.None;
    }

    private void HandleTileSelected(Tile tile)
    {
        _populationLabel.text = tile.CurrentPopulation.ToString();
    }
}
```

**Why this pattern:** Every UI view in the project follows the same lifecycle — `InitializeElements → RegisterCallbacks → UnregisterCallbacks → ShowPanel`. The base class enforces this structure and handles the `UIDocument` setup, so subclasses only focus on their specific UI elements and logic.

> See also: [UnityUIToolkitInstructions.md](UnityUIToolkitInstructions.md) for detailed guidance on data binding, querying elements, and BEM naming.

---

## Singleton Pattern

- ⚠️ Singletons are **discouraged** on this project. If one is genuinely needed, state its usage and the reason it was
  necessary in a comment at the declaration — see
  [Singleton policy](UnityArchitectureInstructions.md#singleton-policy).
- ✅ Before reaching for one, check whether a service registered `Lifetime.Singleton` in the boot scope solves the same
  problem: one instance, globally reachable, injected through a constructor instead of a static field. See
  [Dependency injection: VContainer](UnityArchitectureInstructions.md#dependency-injection-vcontainer).
- ✅ When a Singleton is still justified (e.g. a static bridge that a third-party API requires by contract), use the `_`
  prefix for the mutable static instance field, per the [style guide](../UnityStyleGuide.md#fields).
- ✅ Provide a static `Instance` property and destroy duplicates in `Awake`.
- ✅ Use `DontDestroyOnLoad` to persist the singleton across scene loads only if it has to.
- ✅ Reset the static in a `[RuntimeInitializeOnLoadMethod]` so a destroyed instance isn't kept alive across Play
  sessions when Domain Reload is disabled.
- ✅ Ensure proper cleanup of resources when the singleton is destroyed.

```csharp
// Singleton: the payment plugin's adapter contract requires a static entry point that nothing here
// constructs, so there is no constructor for VContainer to inject into.
// (State the usage and the reason at the declaration, as here.)
public class PaymentCallbackBridge : MonoBehaviour
{
    // Mutable static field: `_` prefix, same as instance fields
    private static PaymentCallbackBridge _instance;

    public static PaymentCallbackBridge Instance => _instance;

    private void Awake()
    {
        // Ensure singleton reference is set and handle duplicates
        if (_instance != null && _instance != this)
        {
            Destroy(gameObject);
            return;
        }
        _instance = this;
        DontDestroyOnLoad(gameObject);
    }

    private void OnDestroy()
    {
        if (_instance == this)
            _instance = null;
    }

    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]
    private static void ResetStatics()
    {
        _instance = null;
    }
}
```

---

## Dependency Injection / Service Locator

- ✅ Default: constructor injection through VContainer. A service is a constructor parameter, registered once in a
  `LifetimeScope` — not something a class pulls from a static registry. Full pattern in
  [Dependency injection: VContainer](UnityArchitectureInstructions.md#dependency-injection-vcontainer).
- ✅ Depend on interfaces (`IAudioService`), not concrete classes — the
  [Dependency Inversion Principle](#dependency-inversion-principle) applied.
- ⚠️ A hand-rolled Service Locator is **legacy** here. It is kept below for reading older code and for a project with
  no DI container. It hides dependencies (any class can resolve anything), fails at runtime rather than at
  construction, and needs manual cleanup per scene.

```csharp
// Constructor injection - the dependencies are visible in the signature
public class RecruitmentController : IRecruitmentController
{
    private readonly IGameResources _gameResources;
    private readonly IGameMapService _gameMapService;

    public RecruitmentController(IGameResources gameResources, IGameMapService gameMapService)
    {
        _gameResources = gameResources;
        _gameMapService = gameMapService;
    }
}

// Registered once, in the scope's Configure
builder.Register<IGameResources, GameResources>(Lifetime.Singleton);
builder.Register<IGameMapService, GameMapService>(Lifetime.Singleton);
builder.Register<IRecruitmentController, RecruitmentController>(Lifetime.Scoped);
```

### Legacy: a hand-rolled Service Locator

If a project has no DI container and uses a locator anyway:

- ✅ Register services during `Awake()` in a centralized injector so they are available by the time `Start()` runs.
- ✅ Resolve dependencies in `Awake()` of consuming classes.
- ✅ Clear the registry in `OnDestroy()` to prevent stale references across scene loads.
- ⚠️ Avoid overusing the Service Locator — it can obscure dependencies if every class resolves everything through it.

**Legacy worked example — a `ServiceLocator` for runtime dependency resolution:**

```csharp
// ServiceLocator.cs — Lightweight service registry (actual project code)
public static class ServiceLocator
{
    private static readonly Dictionary<Type, object> Services = new();

    public static void Register<T>(T service) where T : class
    {
        Services[typeof(T)] = service;
    }

    public static T Resolve<T>() where T : class
    {
        if (Services.TryGetValue(typeof(T), out object service))
        {
            return service as T;
        }
        throw new InvalidOperationException($"Service of type {typeof(T)} not registered.");
    }

    public static void Unregister<T>() where T : class
    {
        Services.Remove(typeof(T));
    }

    public static void Clear()
    {
        Services.Clear();
    }
}
```

**Centralized registration via a DependencyInjector MonoBehaviour:**

```csharp
// DependencyInjector.cs — Registers scene services at startup (actual project code)
public class DependencyInjector : MonoBehaviour
{
    [SerializeField] private GameResources _gameResources;
    [SerializeField] private RecruitmentManager _recruitmentManager;
    [SerializeField] private BuildQueueService _tileConstructionManager;
    [SerializeField] private GameMapController _gameMapController;

    private void Awake()
    {
        ServiceLocator.Register(_gameResources);
        ServiceLocator.Register(_recruitmentManager);
        ServiceLocator.Register(_tileConstructionManager);
        ServiceLocator.Register(_gameMapController);
    }

    private void OnDestroy()
    {
        ServiceLocator.Clear();
    }
}
```

**Resolving dependencies in a consumer:**

```csharp
public class RecruitmentManager : MonoBehaviour
{
    private GameResources _gameResources;
    private GameMapController _gameMapController;

    private void Awake()
    {
        // Resolve dependencies registered by DependencyInjector
        _gameResources = ServiceLocator.Resolve<GameResources>();
        _gameMapController = ServiceLocator.Resolve<GameMapController>();
    }
}
```

---

## Composition over Inheritance

- ✅ Prefer composing GameObjects from multiple focused components rather than building deep inheritance hierarchies.
- ✅ Each component should follow the Single Responsibility Principle — one focused job per MonoBehaviour.
- ✅ Use `GetComponent<T>()` in `Awake()` to wire sibling components on the same GameObject.
- ✅ This approach makes it easy to add, remove, or swap behaviors without modifying existing classes.

**Worked example — composition for a tile system:**

```csharp
// Tile.cs — The primary tile component delegates to focused sub-components
public class Tile : MonoBehaviour
{
    [SerializeField] private TileGarrison _tileMilitary;

    private void Awake()
    {
        // Wire sibling components via GetComponent
        _tileMilitary = GetComponent<TileGarrison>();
    }

    // Tile handles population, happiness, and taxation
    // TileGarrison handles recruitment pools and military strength
    // TileBuildings handles construction and building bonuses
}
```

```csharp
// TileGarrison.cs — Focused on military/recruitment concerns only
public class TileGarrison : MonoBehaviour
{
    private Tile _tile;
    [SerializeField] private int _currentRecruits;
    [SerializeField] private int _newRecruitsPerTurn = 5;

    private void Awake()
    {
        _tile = GetComponent<Tile>();
    }

    private void OnEnable()
    {
        StaticGameEvents.OnTurnEnded += IncreaseRecruitPoolEndOfTurn;
    }

    private void OnDisable()
    {
        StaticGameEvents.OnTurnEnded -= IncreaseRecruitPoolEndOfTurn;
    }

    public void IncreaseRecruitPoolEndOfTurn()
    {
        _currentRecruits += _newRecruitsPerTurn;
    }
}
```

**Tile component architecture:**
```
GameObject: "Tile_Farmland"
├── Tile                  — Population, happiness, taxation
├── TileGarrison          — Recruitment pool, military strength
├── TileBuildings         — Building slots, construction bonuses
└── BuildQueueService — Active construction logic
```

**Why composition:** Each component can be developed, tested, and iterated independently. Adding a new tile concern (e.g., trade routes) means adding a new component — not modifying the existing `Tile` class.

---

## Object Pooling

- ✅ Use object pooling for frequently spawned and destroyed objects (e.g., bullets, enemies, particle effects) to reduce runtime allocations
  and improve performance.
- ✅ Prefer Unity's built-in pooling APIs (e.g., `UnityEngine.Pool.ObjectPool<T>`) in Unity 6 and later, rather than implementing custom
  pooling logic.
- ✅ Also consider `CollectionPool<T>`, `ListPool<T>`, and `DictionaryPool<TKey, TValue>` for reusing temporary collections inside methods — this avoids allocations inside loops.
- ✅ Initialize pools at scene load or on demand, and pre-warm with a reasonable number of objects to avoid spikes during gameplay.
- ✅ Always reset pooled objects' state (position, rotation, active state, etc.) before reusing them.
- ✅ Return objects to the pool instead of destroying them; never use `Destroy()` on pooled objects except during cleanup.
- ✅ Use clear, descriptive method names like `GetFromPool()` and `ReturnToPool()` for pool operations.
- ✅ Keep pool management logic encapsulated — don't expose pool internals to consumers.
- ✅ Use `[DisallowMultipleComponent]` and `[RequireComponent]` as needed to enforce correct usage on pooled objects.
- ❌ Avoid pooling objects with complex or persistent state that is hard to reset.

> See also: [Collection type selection](../UnityStyleGuide.md#collection-type-selection) in the style guide for guidance on avoiding allocations inside loops.

```csharp
// Example: Using Unity's built-in ObjectPool<T>
using UnityEngine.Pool;

public class BulletPool : MonoBehaviour
{
    [SerializeField] private Bullet _bulletPrefab;
    private ObjectPool<Bullet> _pool;

    private void Awake()
    {
        _pool = new ObjectPool<Bullet>(
            createFunc: () => Instantiate(_bulletPrefab),
            actionOnGet: bullet => bullet.gameObject.SetActive(true),
            actionOnRelease: bullet => bullet.gameObject.SetActive(false),
            actionOnDestroy: bullet => Destroy(bullet.gameObject),
            collectionCheck: false,
            defaultCapacity: 20,
            maxSize: 100
        );
    }

    public Bullet GetFromPool()
    {
        return _pool.Get();
    }

    public void ReturnToPool(Bullet bullet)
    {
        _pool.Release(bullet);
    }
}
```

```csharp
// Example: Using CollectionPool to avoid allocations in methods
using UnityEngine.Pool;

public void ProcessNearbyEnemies(Vector3 position, float radius)
{
    // Borrow a list from the pool instead of allocating a new one
    var nearbyEnemies = ListPool<Enemy>.Get();

    try
    {
        FindEnemiesInRadius(position, radius, nearbyEnemies);
        foreach (var enemy in nearbyEnemies)
        {
            enemy.Alert();
        }
    }
    finally
    {
        // Always return the list to the pool
        ListPool<Enemy>.Release(nearbyEnemies);
    }
}
```

---

## Factory Pattern

- ✅ Use the Factory pattern to centralize and encapsulate object creation logic.
- ✅ Useful when the creation process involves setup steps beyond simple instantiation.
- ✅ Keeps the calling code clean by hiding construction details.

```csharp
// Example: Factory method for creating army units from ScriptableObject data
public class ArmyController : MonoBehaviour
{
    [SerializeField] private List<ArmyUnitData> _activeUnits = new();

    public void RecruitUnit(ArmyUnitConfig unitData)
    {
        // Factory logic: create runtime data from static configuration
        var newUnit = new ArmyUnitData(unitData);
        newUnit.CurrentHealth = unitData.Health;
        newUnit.CurrentMorale = 100;
        newUnit.CurrentSquadSize = unitData.SizeSquad;

        _activeUnits.Add(newUnit);
    }
}
```

```csharp
// Example: A more formal factory for spawning GameObjects
public class EnemyFactory : MonoBehaviour
{
    [SerializeField] private GameObject _infantryPrefab;
    [SerializeField] private GameObject _cavalryPrefab;
    [SerializeField] private GameObject _archerPrefab;

    public GameObject CreateEnemy(UnitCategory category, Vector3 spawnPosition)
    {
        GameObject prefab = category switch
        {
            UnitCategory.Infantry => _infantryPrefab,
            UnitCategory.Cavalry  => _cavalryPrefab,
            UnitCategory.Ranged   => _archerPrefab,
            _ => throw new ArgumentException($"Unknown unit category: {category}")
        };

        var enemy = Instantiate(prefab, spawnPosition, Quaternion.identity);
        enemy.name = $"{category}_{Time.frameCount}";
        return enemy;
    }
}
```

---

## Command Pattern

- ⚠️ Consider the Command pattern for input handling, undo/redo, action history, and replay systems.
- ✅ Encapsulates a request as an object, allowing parameterization, queuing, and logging of operations.
- ✅ Pair with a `Stack<ICommand>` for undo/redo functionality.

```csharp
// Command interface
public interface ICommand
{
    void Execute();
    void Undo();
}

// Concrete command: move an army
public class MoveArmyCommand : ICommand
{
    private readonly ArmyController _army;
    private readonly Vector3 _targetPosition;
    private Vector3 _previousPosition;

    public MoveArmyCommand(ArmyController army, Vector3 targetPosition)
    {
        _army = army;
        _targetPosition = targetPosition;
    }

    public void Execute()
    {
        _previousPosition = _army.transform.position;
        _army.transform.position = _targetPosition;
    }

    public void Undo()
    {
        _army.transform.position = _previousPosition;
    }
}
```

```csharp
// Command invoker with undo/redo stacks
public class CommandInvoker
{
    private readonly Stack<ICommand> _undoStack = new();
    private readonly Stack<ICommand> _redoStack = new();

    public void ExecuteCommand(ICommand command)
    {
        command.Execute();
        _undoStack.Push(command);
        _redoStack.Clear();
    }

    public void Undo()
    {
        if (_undoStack.Count == 0) return;

        var command = _undoStack.Pop();
        command.Undo();
        _redoStack.Push(command);
    }

    public void Redo()
    {
        if (_redoStack.Count == 0) return;

        var command = _redoStack.Pop();
        command.Execute();
        _undoStack.Push(command);
    }
}
```

---

## Strategy Pattern

- ⚠️ Consider the Strategy pattern when you need interchangeable behaviors that can be swapped at runtime.
- ✅ Define a common interface for the behavior, then create concrete implementations for each variation.
- ✅ Useful for AI behavior, movement types, attack styles, or tax calculation strategies.

```csharp
// Strategy interface
public interface IMovementStrategy
{
    void Move(Transform transform, Vector3 target, float speed);
}

// Concrete strategies
public class DirectMovement : IMovementStrategy
{
    public void Move(Transform transform, Vector3 target, float speed)
    {
        transform.position = Vector3.MoveTowards(transform.position, target, speed * Time.deltaTime);
    }
}

public class PatrolMovement : IMovementStrategy
{
    private readonly Vector3[] _waypoints;
    private int _currentWaypointIndex;

    public PatrolMovement(Vector3[] waypoints)
    {
        _waypoints = waypoints;
    }

    public void Move(Transform transform, Vector3 target, float speed)
    {
        var waypoint = _waypoints[_currentWaypointIndex];
        transform.position = Vector3.MoveTowards(transform.position, waypoint, speed * Time.deltaTime);

        if (Vector3.Distance(transform.position, waypoint) < 0.1f)
        {
            _currentWaypointIndex = (_currentWaypointIndex + 1) % _waypoints.Length;
        }
    }
}
```

```csharp
// Context: unit uses a strategy that can be swapped at runtime
public class ArmyMovementController : MonoBehaviour
{
    [SerializeField] private float _moveSpeed = 5f;

    private IMovementStrategy _movementStrategy;
    private Vector3 _targetPosition;

    public void SetMovementStrategy(IMovementStrategy strategy)
    {
        _movementStrategy = strategy;
    }

    private void Update()
    {
        _movementStrategy?.Move(transform, _targetPosition, _moveSpeed);
    }
}
```

---

## Troubleshooting

**A singleton is null after reloading the scene that created it.**
The instance field is static and survived, but the object it pointed at was destroyed. Null-check
against Unity's overloaded `==` (which reports destroyed objects as null) and re-create, or move the
object to a bootstrap scene that never unloads. See
[Scenes & Lifecycle](UnityScenesAndLifecycleInstructions.md#dontdestroyonload-discipline).

**A singleton is null on the second Play with Domain Reload disabled.**
The static field kept a reference to a destroyed object. Reset it in a
`[RuntimeInitializeOnLoadMethod]`.

**An event fires twice.**
Subscribed in both `Awake` and `OnEnable`, so re-enabling adds a second handler. Subscribe only in
`OnEnable` and unsubscribe in `OnDisable`. The R3 version of the same bug is `.AddTo(this)` inside
`OnEnable`: it isn't undone on disable, so each re-enable adds a subscription. Subscribe in `Start`, or clear
a `CompositeDisposable` in `OnDisable`.

**`ObjectDisposedException` from a Subject.**
The owner disposed it in `OnDestroy`, and something raised or subscribed afterwards. Only the owner should call
`OnNext`, and only while it is alive; a subscriber that may outlive the publisher must not subscribe to it late.

**An event handler runs on a destroyed object.**
A missed unsubscribe on a static or long-lived event. The publisher is holding the delegate — and
therefore the object — alive.

**A state machine gets stuck in one state.**
The transition condition is evaluated in the state's `Update` but the state never re-checks it, or
`Exit`/`Enter` are called in the wrong order. Log every transition while debugging; it is almost
always visible immediately.

**Object pool returns objects that are still active, or runs dry.**
Something took an object and never released it. Set `collectionCheck: true` in development — it
throws when an object is released twice, which usually reveals the leak.

**A service resolves to null through the legacy Service Locator.**
Registration order. The consumer's `Awake` ran before the provider's. Register in a bootstrap scene
that loads first, or resolve lazily in `Start` rather than `Awake`. VContainer doesn't have this failure
mode — a missing registration throws a `VContainerException` naming the type instead of returning null.

## Additional Resources

- [Level Up Your Code with Design Patterns and SOLID](https://unity.com/resources/design-patterns-solid-ebook) — Unity ebook
- [UnityStyleGuide.md](../UnityStyleGuide.md) — C# style guide, naming conventions, and coding patterns
- [UnityArchitectureInstructions.md](UnityArchitectureInstructions.md) — MVC layering, VContainer, Singleton policy, MessagePipe, R3
- [UnityUIToolkitInstructions.md](UnityUIToolkitInstructions.md) — UI Toolkit reference including data binding and MVP pattern
- [Game Programming Patterns](https://gameprogrammingpatterns.com/) — Robert Nystrom's free online book
