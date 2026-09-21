# Unity ScriptableObjects

> **General Unity best practice, plus a few marked preferences.** Rules tagged `[opinion]` are my
> stylistic choices and are also listed in the
> [opinion table](../UnityStyleGuide.md#whats-opinionated-here) of `UnityStyleGuide.md`. Everything
> else is standard Unity practice. Project-specific settings live in
> [`UnityCustomInstructions/`](../UnityCustomInstructions/).

ScriptableObjects as authored, shared, serialized data: when to use one, how to name it, how to make
it creatable from the Project window, and how to keep it from turning into a runtime-state bug.

Table of contents:
- [When to use one](#when-to-use-one)
- [Naming](#naming)
- [CreateAssetMenu](#createassetmenu)
- [Fields and properties](#fields-and-properties)
- [Config versus runtime data](#config-versus-runtime-data)
- [Organising assets](#organising-assets)
- [Large assets: binary serialization](#large-assets-binary-serialization)
- [Odin Inspector](#odin-inspector)
- [Review checklist](#review-checklist)
- [Learn more](#learn-more)

---

## When to use one

- ✅ Favor ScriptableObjects for static configuration data and reusable content that stays the same
  while the game runs: weapons, enemy stats, skill effects, sound definitions, feature settings.
- ❌ Don't use them to store data that changes during gameplay — player health, score, inventory
  contents. An edit made to an asset in Play Mode **persists after you stop**, so a runtime write
  silently rewrites authored data. See [Config versus runtime data](#config-versus-runtime-data).
- ✅ Use them to reduce coupling between systems. Feed configuration into MonoBehaviours (via a
  `[SerializeField]` reference) or into pure C# classes (via their constructor) instead of having
  them fetch data manually.
- ✅ Keep each ScriptableObject focused on a single responsibility to enhance reusability and
  maintainability.
- ✅ Keep data and logic separate. A ScriptableObject should primarily hold data; only add logic that
  directly relates to that data (validation, a derived value).

```csharp
// MonoBehaviour - config assigned in the Inspector
public class EnemySpawner : MonoBehaviour
{
    [SerializeField] private EnemyConfig _config;
}

// Pure C# class - config passed in through the constructor
public class EnemyController
{
    private readonly EnemyConfig _config;

    public EnemyController(EnemyConfig config)
    {
        _config = config;
    }
}
```

---

## Naming

- ✅ End every ScriptableObject class name with **`Config`**: `WeaponConfig`, `EnemyConfig`,
  `LeaderboardConfig`. `[opinion]` ScriptableObjects hold authored, static data, so "config"
  describes all of them, and the suffix makes the type recognisable at a glance in code and in
  Project-window search.
- ✅ One ScriptableObject class per file, and the file name matches the class name exactly. Unity
  needs the match to create and serialize the asset.
- ✅ Asset files are PascalCase and named after what they configure: `GoblinConfig.asset`
  (an instance of `EnemyConfig`).

```csharp
// EnemyConfig.cs
public class EnemyConfig : ScriptableObject { }   // class name = file name = "EnemyConfig"

// Instances live in the Project window:
//   GoblinConfig.asset, OrcConfig.asset
```

---

## CreateAssetMenu

- ✅ Always mark ScriptableObjects with `[CreateAssetMenu]` for easy asset creation via the Project
  window. Don't make designers create assets through code (`ScriptableObject.CreateInstance` behind
  a custom menu) when the attribute does it for you.
- ✅ `menuName` follows one path template: **`"<Category>/<Asset Name>"`**. `[opinion]`
  - `<Category>` is the feature area the config belongs to (`Enemies`, `Inventory`, `Audio`). Use the
    same name as the feature's folder so the two line up.
  - `<Asset Name>` is human-readable, with spaces (`Enemy Config`), even though the class and file
    names are PascalCase without them.
  - Every feature gets its own entry under **Assets ▸ Create** instead of custom types scattering
    across Unity's defaults.
- ✅ `fileName` is the default filename offered on creation. Set it to the **class name exactly**
  (PascalCase, no spaces) so assets stay easy to find and search.

```csharp
[CreateAssetMenu(fileName = "EnemyConfig", menuName = "Enemies/Enemy Config")]
public class EnemyConfig : ScriptableObject
{
    [SerializeField] private int _health;
    [SerializeField] private float _moveSpeed;

    public int Health => _health;
    public float MoveSpeed => _moveSpeed;
}

[CreateAssetMenu(fileName = "ItemConfig", menuName = "Inventory/Item Config")]
public class ItemConfig : ScriptableObject
{
    [SerializeField] private string _displayName;
    [SerializeField] private Sprite _icon;

    public string DisplayName => _displayName;
    public Sprite Icon => _icon;
}
```

---

## Fields and properties

- ✅ Use properties to expose data from ScriptableObjects instead of public fields, for better
  encapsulation. Keep the values in `[SerializeField] private` fields and expose them through
  **get-only public properties**: consumers read the value, and nothing outside the asset can
  overwrite it.
- ❌ Never public fields, and never serialize a property directly. See
  [Properties](../UnityStyleGuide.md#properties) in the style guide.

```csharp
// Good - consumers can read Health but not change it
[SerializeField] private int _health;
public int Health => _health;

// Bad - anything can rewrite the authored value
public int health;
```

---

## Config versus runtime data

A ScriptableObject is **authored, feature-wide configuration**. Data that belongs to one entity at
runtime is a different thing and should not be a ScriptableObject.

| | Config | Runtime data |
|---|---|---|
| Example | `LeaderboardConfig` (max entries, refresh interval) | `LeaderboardEntry` (player name, score, rank) |
| Represented as | ScriptableObject | Plain C# `struct` or `class` |
| Created by | A designer, in the Editor | Code, at runtime |
| Changes at runtime | Never | Constantly |

```csharp
// Authored, feature-wide settings
[CreateAssetMenu(fileName = "LeaderboardConfig", menuName = "Leaderboard/Leaderboard Config")]
public class LeaderboardConfig : ScriptableObject
{
    [SerializeField] private int _maxEntries = 100;
    [SerializeField] private float _refreshIntervalInSeconds = 30f;

    public int MaxEntries => _maxEntries;
    public float RefreshIntervalInSeconds => _refreshIntervalInSeconds;
}

// One entry's runtime data - a plain struct, not an asset
public struct LeaderboardEntry
{
    public string PlayerName;
    public int Score;
    public int Rank;
}
```

- ⚠️ If a component genuinely needs a mutable copy of a config, `Instantiate` it and destroy the
  copy when you're done. Writing to the shared asset changes it for every user of it, and in the
  Editor it changes the asset on disk.
- ℹ️ The symptoms of getting this wrong (changes persisting after Play Mode, several objects
  unexpectedly sharing state) are diagnosed in
  [Debugging](UnityDebuggingInstructions.md#scriptableobject-runtime-issues).

```csharp
[SerializeField] private WeaponConfig _config;    // shared, read-only
private WeaponConfig _runtimeConfig;              // this object's own copy

private void Awake()
{
    _runtimeConfig = Instantiate(_config);
}

private void OnDestroy()
{
    if (_runtimeConfig != null)
    {
        Destroy(_runtimeConfig);
    }
}
```

---

## Organising assets

- ✅ Store ScriptableObject assets in a dedicated folder structure, e.g. `Assets/Data/Weapons/`,
  grouped by feature. The `<Category>` in `menuName` should match the feature folder name.
- ❌ Don't put them in a `Resources/` folder to load them by string. Reference them with
  `[SerializeField]` or Addressables. See
  [Assets & Memory](UnityAssetsAndMemoryInstructions.md).

---

## Large assets: binary serialization

- ✅ For a large data asset, add `[PreferBinarySerialization]` to the class. The `.asset` file is then
  stored as binary instead of YAML text, which reads and writes noticeably faster at size.
- ⚠️ Binary assets can't be diffed or merged in version control. Reserve it for assets that are
  either large or rarely hand-edited — not for small configs a person tweaks and reviews in pull
  requests.
- ⚠️ The class name must match the `.cs` file name exactly, or a `[PreferBinarySerialization]` asset
  can fail to serialize correctly. This is one more reason the [naming rule](#naming) matters
  beyond readability.

```csharp
[PreferBinarySerialization]
[CreateAssetMenu(fileName = "LevelDataConfig", menuName = "Levels/Level Data Config")]
public class LevelDataConfig : ScriptableObject
{
    [SerializeField] private float[] _heightSamples;

    public IReadOnlyList<float> HeightSamples => _heightSamples;
}
```

---

## Odin Inspector

Only relevant if the project uses Odin Inspector.

- ✅ `SerializedScriptableObject` is only needed when a field's type isn't natively serializable by
  Unity — interfaces, dictionaries, generics. Otherwise derive from plain `ScriptableObject`.
- ℹ️ A private field with no `[SerializeField]` is already skipped by both Unity's and Odin's
  serializers. `[System.NonSerialized]` on it is redundant, not required.

---

## Review checklist

| Check | Look for |
|---|---|
| Naming | ScriptableObject class not ending in `Config` |
| Naming | File name that doesn't match the class name |
| Menu | Missing `[CreateAssetMenu]` on a designer-authored asset |
| Menu | `menuName` not in `"<Category>/<Asset Name>"` form, or `fileName` not equal to the class name |
| Encapsulation | `public` fields instead of `[SerializeField] private` plus a get-only property |
| Runtime state | Writes to a shared asset outside `OnValidate`/Editor code |
| Runtime state | Per-entity data modelled as a ScriptableObject instead of a plain struct/class |
| Design | One asset type mixing unrelated settings, or holding logic unrelated to its data |
| Coupling | Components looking up their own configuration instead of receiving it |
| Loading | Asset placed in `Resources/` and loaded by string |
| Serialization | `[PreferBinarySerialization]` on a small asset that people edit and review in PRs |

---

## Learn more

- [ScriptableObject, Unity manual](https://docs.unity3d.com/6000.3/Documentation/Manual/class-ScriptableObject.html)
- [CreateAssetMenuAttribute scripting reference](https://docs.unity3d.com/6000.3/Documentation/ScriptReference/CreateAssetMenuAttribute.html)
- [PreferBinarySerialization scripting reference](https://docs.unity3d.com/6000.3/Documentation/ScriptReference/PreferBinarySerialization.html)
