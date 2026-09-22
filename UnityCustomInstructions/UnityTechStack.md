# Project Tech Stack

> **Project-specific. You are expected to edit this file.**
> This is the first file to change when you adopt this guide. Everything here tells your coding
> assistant which Unity APIs are valid for *your* project — get it wrong and it will confidently
> generate code for the wrong render pipeline or input system.

## Versions

- ℹ️ This project uses **Unity 6.3 (6000.3.x)**. Use sources and documentation that apply to Unity 6
  or later. Do not generate code for Unity 2022 or earlier.
- ℹ️ C# language version: **C# 9.0**. Target-typed `new()`, records, and pattern matching enhancements
  are available; newer language features are not.

## Systems in use

| Area | This project uses | Not used — do not generate |
|---|---|---|
| Input | Input System package | Legacy Input Manager (`Input.GetAxis`, `Input.GetKey`) |
| UI | uGUI (Canvas/Image/TextMeshPro) | UI Toolkit (UXML/USS), IMGUI for runtime UI |
| Rendering | Universal Render Pipeline (URP 17.3) | Built-in Render Pipeline, HDRP |
| Async | UniTask (Cysharp) | `Awaitable`, coroutines except where per-frame iteration is genuinely needed |
| Dependency injection | VContainer — constructor injection, one root `LifetimeScope` per scene | Hand-rolled `ServiceLocator`, manually `new`ing services |
| Messaging | MessagePipe — reserved for narrow cases only (see the Architecture guide) | A default/global event bus for everyday cross-system communication |
| Reactive callbacks | R3 (`Observable`), mainly for deterministic subscription cleanup via `AddTo()` | UniRx (legacy) |
| Pooling | `UnityEngine.Pool.ObjectPool<T>` | Hand-rolled pool implementations |

## Conventions that follow from the stack

- ℹ️ Prefer UniTask over `Awaitable` or coroutines for async gameplay code. `Awaitable` is still valid
  Unity 6 API and works the same way if a project isn't on UniTask, but it isn't this project's default.
  See `UnityReferenceGuides/UnityUniTaskInstructions.md` for common patterns.
- ℹ️ When instantiating frequently, favour `UnityEngine.Pool.ObjectPool<T>` with
  `actionOnGet`/`actionOnRelease` to toggle active state.
- ℹ️ UI work goes through uGUI. See
  [UnityUGUIInstructions.md](../UnityReferenceGuides/UnityUGUIInstructions.md).
  If you switch to UI Toolkit, read
  [UnityUIToolkitInstructions.md](../UnityReferenceGuides/UnityUIToolkitInstructions.md) instead and
  update the table above.
- ℹ️ Services are registered with and resolved through VContainer, not constructed manually or
  accessed through statics. Default to a direct, constructor-injected interface reference for
  cross-system communication — VContainer covers this project's needs on its own in the large
  majority of cases. MessagePipe is an optional add-on, installed only when one of the narrow cases
  the Architecture guide lists actually comes up. See
  `UnityReferenceGuides/UnityArchitectureInstructions.md` for the full pattern.

## Packages

List the packages your project depends on so your assistant doesn't suggest APIs you haven't
installed, or reinvent something a package already provides.

| Package | Version | Used for |
|---|---|---|
| `com.unity.inputsystem` | 1.x | All player input |
| `com.unity.render-pipelines.universal` | 17.3 | Rendering |
| `com.unity.addressables` | — | *(fill in or remove)* |
| `com.unity.test-framework` | — | *(fill in or remove)* |
| `com.unity.textmeshpro` | — | Text rendering (uGUI) |
| `com.cysharp.unitask` | — | Async/await for gameplay code |
| `jp.hadashikick.vcontainer` | — | Dependency injection |
| MessagePipe | — | *(optional — install only if/when one of the narrow cases in the Architecture guide comes up; VContainer + direct references handle everything else)* |
| R3 | — | Reactive `Observable` streams |
| Odin Inspector | — | *(optional — fill in if used, remove if not)* |

## Platform targets

- **Primary:** *(e.g. Windows/macOS standalone)*
- **Secondary:** *(e.g. Android)*
- ℹ️ Performance budgets and platform quirks differ a lot between these. Note anything
  non-obvious here — for example a 60 fps cap on mobile, or a memory ceiling.
