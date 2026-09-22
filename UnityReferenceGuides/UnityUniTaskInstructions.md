# UniTask

> **Applies once a project has adopted UniTask** — this project's default async approach, per
> [`UnityTechStack.md`](../UnityCustomInstructions/UnityTechStack.md). If a project isn't on UniTask,
> Unity's built-in `Awaitable` works the same way for most of the patterns below, just without the
> UniTask-specific types (`UniTaskVoid`, `UniTaskCompletionSource`, `PlayerLoopTiming`). Personal style
> preferences live in [`UnityStyleGuide.md`](../UnityStyleGuide.md).

[UniTask](https://github.com/Cysharp/UniTask) (Cysharp) is a zero-allocation-where-possible async/await
implementation for Unity. This guide covers the patterns that come up in everyday gameplay code — for
the exhaustive API reference, go to the source repo linked in [Learn more](#learn-more) rather than
treating this as a substitute for it.

Table of contents:
- [Why UniTask](#why-unitask)
- [Naming](#naming)
- [Cancellation](#cancellation)
- [Error handling](#error-handling)
- [Don't mark a method async if it doesn't need to be](#dont-mark-a-method-async-if-it-doesnt-need-to-be)
- [Fire-and-forget: UniTaskVoid](#fire-and-forget-unitaskvoid)
- [Composing tasks: WhenAll / WhenAny](#composing-tasks-whenall-whenany)
- [Timing control: PlayerLoopTiming](#timing-control-playerlooptiming)
- [Closure-free overloads](#closure-free-overloads)
- [Delays](#delays)
- [Bridging callbacks: UniTaskCompletionSource](#bridging-callbacks-unitaskcompletionsource)
- [Interop with Task](#interop-with-task)
- [Async enumerables](#async-enumerables)
- [When a coroutine is still the right call](#when-a-coroutine-is-still-the-right-call)
- [Common pitfalls](#common-pitfalls)
- [Review checklist](#review-checklist)
- [Learn more](#learn-more)

---

## Why UniTask

- ✅ UniTask avoids capturing `SynchronizationContext`/`ExecutionContext` on every `await`, which the
  built-in `Task` pays as a real per-`await` cost. That's the actual reason to prefer it for gameplay
  async code — not a raw allocation win over a plain C# delegate. A `UniTask.Subscribe`-style
  `Subscribe()` call still allocates; the win is in what happens around every `await`, at high
  `await`-frequency (per-frame loops, frequent short-lived operations).
- ✅ It plays natively with Unity's player loop (`PlayerLoopTiming`), so you can await a specific point
  in the frame (`Update`, `LateUpdate`, `FixedUpdate`, end of frame) without a `MonoBehaviour` driving
  a coroutine underneath.
- ℹ️ It is a separate type from `Task` and from `Awaitable`. They interoperate (see
  [Interop with Task](#interop-with-task)), but a method that returns `UniTask` doesn't implicitly
  work anywhere a `Task` or `Awaitable` is expected.

---

## Naming

- ✅ Prefix async methods with **`Task`**, never suffix with `Async` — `TaskInitialize`,
  `TaskSpawnEnemy`, `TaskLoadLevel`. `[opinion]`
- ⚠️ This differs from the `Async`-suffix convention used elsewhere in this repo's examples, which
  predate the switch to UniTask. Treat `Task`-prefix as the naming rule going forward; older
  `Async`-suffixed examples in other guides reflect the previous convention until they're updated.
- ✅ Discarded lambda/callback parameters are named `_`, the same as anywhere else in this style
  guide — not to be confused with the `Task` naming prefix, which is a method-name convention, not a
  parameter one.

```csharp
public class EnemySpawner : MonoBehaviour
{
    private int _activeEnemyCount;

    public async UniTask TaskSpawnWave(int enemyCount)
    {
        for (int i = 0; i < enemyCount; i++)
        {
            await TaskSpawnEnemy();
        }
    }

    private async UniTask TaskSpawnEnemy()
    {
        await UniTask.Delay(500);
        _activeEnemyCount++;
    }
}
```

---

## Cancellation

- ✅ On a `MonoBehaviour`, pull the cancellation token from `this.GetCancellationTokenOnDestroy()`
  rather than managing a `CancellationTokenSource` by hand. Async work is then auto-cancelled when the
  component is destroyed, without a matching `OnDestroy`/`Dispose` pair to remember.
- ✅ Pass the token through to every `await` in the method — a token captured once at the top and
  never threaded through doesn't cancel anything downstream.
- ⚠️ Manage a `CancellationTokenSource` manually only for genuinely special cases: cancelling
  independently of the component's lifetime (a skippable cutscene, a timeout shorter than the
  component's lifetime), or cancellation that needs to be triggered from outside the component.

```csharp
public class LoadingScreen : MonoBehaviour
{
    public async UniTask TaskFadeOut()
    {
        await UniTask.Delay(1000, cancellationToken: this.GetCancellationTokenOnDestroy());
    }
}
```

---

## Error handling

- ✅ Wrap every async method body in `try/catch (System.Exception e)` — applied consistently, not
  just in the methods where there's fallback logic to run. `[opinion]` This is stricter than this
  guide's general try/catch stance (reserved for genuinely external failures — see
  [Using try-catch & debugger breaks](../UnityStyleGuide.md#using-try-catch--debugger-breaks)); async
  methods get the wrap unconditionally, because an unhandled exception inside `UniTask` can otherwise
  surface far from where it actually happened, or get silently lost depending on how the task is
  awaited.
- ✅ Re-throw cancellation explicitly, before the general catch: `if (e is
  System.OperationCanceledException) throw;` — a cancellation isn't a failure, and swallowing it here
  breaks the caller's ability to observe that the operation was cancelled rather than completed.
- ✅ Route caught exceptions through this project's centralized logging/exception-handling utility
  (see [Debugging](../UnityStyleGuide.md#debugging)) rather than swallowing them silently.
- ✅ Use `finally` for cleanup that must run regardless of outcome — closing a loading spinner,
  stopping a timer.
- ✅ Prefer a guard clause over a nested conditional at the top of the method: `if (data == null)
  return;`, before the `try` block if the check doesn't need to be inside it.

```csharp
public async UniTask TaskLoadPlayerData()
{
    try
    {
        PlayerData data = await TaskFetchFromServer();
        if (data == null) return;

        ApplyPlayerData(data);
    }
    catch (System.OperationCanceledException)
    {
        throw;
    }
    catch (System.Exception e)
    {
        // AppLogger is a placeholder for this project's logging/exception utility
        AppLogger.LogException(e);
    }
    finally
    {
        HideLoadingSpinner();
    }
}
```

---

## Don't mark a method async if it doesn't need to be

- ⚠️ The compiler generates the full state machine for an `async` method regardless of whether an
  `await` inside it actually suspends. A method that usually completes synchronously still pays that
  overhead on every call if it's marked `async`.
- ✅ Split the synchronous fast path out from the genuinely asynchronous path, instead of wrapping
  both in one `async` method that awaits conditionally.

```csharp
// Avoid - pays async overhead even on the fast path
public async UniTask<Sprite> TaskGetIconAsync(string id)
{
    if (_cache.TryGetValue(id, out Sprite cached))
    {
        return cached;   // still went through the state machine to get here
    }

    return await TaskLoadIconFromDisk(id);
}

// Better - the common case never touches the state machine
public UniTask<Sprite> TaskGetIcon(string id)
{
    if (_cache.TryGetValue(id, out Sprite cached))
    {
        return UniTask.FromResult(cached);
    }

    return TaskLoadIconFromDisk(id);
}
```

---

## Fire-and-forget: UniTaskVoid

- ✅ Use `UniTaskVoid` (not `async void`) for a method that's genuinely fire-and-forget — a callback
  handler, an event subscriber — where the caller isn't going to `await` the result.
- ✅ Call `.Forget()` on it at the call site. Forgetting to call `.Forget()` is the single most common
  UniTask mistake — see [Common pitfalls](#common-pitfalls).
- ⚠️ `UniTaskVoid` still needs the same error handling as any other async method — an unobserved
  exception inside one doesn't just disappear, it surfaces through UniTask's global exception handler,
  which is not the same as handling it at the call site.

```csharp
private void OnEnable()
{
    _jumpButton.onClick.AddListener(HandleJumpClicked);
}

private void HandleJumpClicked()
{
    TaskPlayJumpSequence().Forget();
}

private async UniTaskVoid TaskPlayJumpSequence()
{
    try
    {
        await TaskPlayJumpAnimation();
        await TaskSpawnLandingParticles();
    }
    catch (System.Exception e)
    {
        AppLogger.LogException(e);
    }
}
```

---

## Composing tasks: WhenAll / WhenAny

- ✅ `UniTask.WhenAll(...)` to run independent async operations concurrently and continue once all of
  them finish — loading several assets in parallel, waiting on multiple in-flight requests.
- ✅ `UniTask.WhenAny(...)` for a race — the first one to finish wins, e.g. a timeout raced against
  the actual operation.
- ⚠️ `WhenAll` propagates the first exception it sees but still lets every task run to completion
  first; it doesn't cancel the others just because one failed. Cancel them explicitly via a shared
  `CancellationTokenSource` if failing fast matters.

```csharp
public async UniTask TaskPreloadLevel()
{
    await UniTask.WhenAll(
        TaskLoadTerrain(),
        TaskLoadEnemyConfigs(),
        TaskLoadAudioBank());
}

public async UniTask<bool> TaskFetchWithTimeout(float timeoutSeconds, CancellationToken token)
{
    var (hasResult, _) = await UniTask.WhenAny(
        TaskFetchFromServer(token),
        UniTask.Delay(System.TimeSpan.FromSeconds(timeoutSeconds), cancellationToken: token));

    return hasResult;
}
```

---

## Timing control: PlayerLoopTiming

- ✅ `UniTask.Yield(PlayerLoopTiming.X)` or `UniTask.NextFrame(PlayerLoopTiming.X)` to resume at a
  specific point in Unity's player loop, instead of assuming "next frame" means one specific moment.
- ✅ Default (`PlayerLoopTiming.Update`) matches `MonoBehaviour.Update` timing. Use
  `PlayerLoopTiming.FixedUpdate` to resume alongside physics, `PostLateUpdate` to resume after
  `LateUpdate` (camera-follow-adjacent work), and so on.
- ℹ️ This is UniTask's equivalent of choosing which Unity lifecycle method your logic belongs in — the
  same reasoning from [MonoBehaviour methods](../UnityStyleGuide.md#monobehaviour-methods) about
  `Update` vs `FixedUpdate` vs `LateUpdate` applies to picking a `PlayerLoopTiming`.

```csharp
public async UniTask TaskWaitForPhysicsStep(CancellationToken token)
{
    await UniTask.Yield(PlayerLoopTiming.FixedUpdate, token);
}
```

---

## Closure-free overloads

- ✅ `UniTask.WaitUntil`/`WaitWhile` have a generic `state` overload specifically so the predicate can
  be a `static` lambda instead of capturing an outer variable or `this`. `UniTask.WaitUntilValueChanged`
  is closure-free by construction — `target` is always a required parameter, passed to
  `monitorFunction` as an argument rather than captured.
- ⚠️ The closure-free win only holds if the lambda is actually `static`. `UniTask.WaitUntil(this, x =>
  x._isReady)` without `static` still compiles and still avoids capturing `this` specifically, but a
  non-`static` lambda that references any *other* outer variable brings a closure back — the `state`
  parameter only covers the one value threaded through it.
- ℹ️ This matters here specifically because these are polling waits: the predicate runs on every
  matching `PlayerLoopTiming` tick until it's satisfied, so a per-poll allocation is a repeated cost,
  not a one-off. See
  [Avoiding Per-Frame Allocations](UnityPerformanceOptimizationInstructions.md#avoiding-per-frame-allocations)
  for the general rule this is one specific case of.

```csharp
// Allocates a closure - the lambda captures `this`
await UniTask.WaitUntil(() => _isReady);

// Closure-free - state passed explicitly, lambda can be static
await UniTask.WaitUntil(this, static self => self._isReady);

// WaitWhile follows the same shape
await UniTask.WaitWhile(this, static self => self._isLoading);

// WaitUntilValueChanged is closure-free by construction - target is always explicit
int health = await UniTask.WaitUntilValueChanged(this, static self => self._health);
```

---

## Delays

- ✅ `UniTask.Delay(milliseconds, ...)` for real-time delays; `UniTask.DelayFrame(frameCount, ...)`
  for a fixed number of frames, independent of `Time.timeScale`.
- ✅ Pass `DelayType.UnscaledDeltaTime` explicitly when a delay must ignore `Time.timeScale` (a pause
  menu countdown, UI feedback that should still play while gameplay is paused).
- ❌ Don't build a delay out of a manual `while` loop accumulating `Time.deltaTime` — `UniTask.Delay`
  already does this correctly, including the unscaled-time case.

```csharp
public async UniTask TaskShowToast(string message, CancellationToken token)
{
    _toastLabel.text = message;
    _toastLabel.gameObject.SetActive(true);

    await UniTask.Delay(2000, DelayType.UnscaledDeltaTime, cancellationToken: token);

    _toastLabel.gameObject.SetActive(false);
}
```

---

## Bridging callbacks: UniTaskCompletionSource

- ✅ Use `UniTaskCompletionSource<T>` to wrap a callback-based API (a plugin, a platform SDK, a
  legacy event) into something `await`-able, the same role `TaskCompletionSource<T>` plays for `Task`.
- ✅ Complete it exactly once. Calling `TrySetResult`/`TrySetException`/`TrySetCanceled` more than
  once on the same source is a bug the `Try*` naming makes easy to miss — check the return value in
  anything other than the obvious single-callback case.

```csharp
public UniTask<PurchaseResult> TaskPurchaseItem(string productId)
{
    var completionSource = new UniTaskCompletionSource<PurchaseResult>();

    _store.OnPurchaseComplete += result => completionSource.TrySetResult(result);
    _store.OnPurchaseFailed += error => completionSource.TrySetException(new System.Exception(error));

    _store.InitiatePurchase(productId);
    return completionSource.Task;
}
```

---

## Interop with Task

- ✅ `.AsUniTask()` on a `Task`/`Task<T>` to bring third-party-library async code into a UniTask
  workflow. `.AsTask()` on a `UniTask`/`UniTask<T>` for the reverse, when a library expects a `Task`.
- ⚠️ Converting doesn't remove `Task`'s `SynchronizationContext` capture cost — it happened before the
  conversion, inside the `Task`-returning call. Interop is for boundary-crossing, not a way to make
  third-party `Task`-based code as cheap as native UniTask code.

```csharp
public async UniTask TaskLoadRemoteConfig()
{
    // ThirdPartySdk.FetchAsync() returns Task<string> - bridge it once at the boundary
    string json = await ThirdPartySdk.FetchAsync().AsUniTask();
    ApplyRemoteConfig(json);
}
```

---

## Async enumerables

- ✅ `UniTaskAsyncEnumerable`/`IUniTaskAsyncEnumerable<T>` for a genuinely streamed sequence of
  async values over time — repeated server-sent events, a polling stream — consumed with
  `await foreach`.
- ❌ Don't reach for it to express "a list of things to process one at a time." A plain `foreach` over
  an already-available collection, `await`ing inside the loop body, does that without the extra type.

```csharp
private async UniTaskVoid TaskConsumeServerEvents(CancellationToken token)
{
    await foreach (ServerEvent evt in _connection.SubscribeEvents().WithCancellation(token))
    {
        HandleServerEvent(evt);
    }
}
```

---

## When a coroutine is still the right call

- ✅ UniTask is the default for async gameplay code, but a small number of cases are still more
  naturally expressed as a coroutine — genuinely per-frame iteration driven by Unity's own
  `WaitForFixedUpdate`/`WaitUntil` idioms, or code that has to interoperate with a third-party API
  that only accepts an `IEnumerator`.
- ✅ Name the rare legitimate coroutine with a **`Routine`** suffix — `FadeOutRoutine`,
  `LoadAssetsRoutine` — rather than the `Task`-prefix used for UniTask methods, so the two are never
  visually confused. `[opinion]`
- ❌ Don't mix a coroutine and a UniTask `await` chain inside the same operation. Pick one per
  workflow — converting back and forth mid-operation is a common source of the "it completed twice" /
  "it never completed" class of bug.

```csharp
private IEnumerator FadeOutRoutine()
{
    // Cache the wait - don't allocate one per iteration
    var wait = new WaitForSeconds(0.1f);

    while (_canvasGroup.alpha > 0f)
    {
        _canvasGroup.alpha -= 0.1f;
        yield return wait;
    }
}
```

---

## Common pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| Forgot `.Forget()` on a `UniTaskVoid` call | Compiler warning ignored; exceptions inside it vanish silently or surface far from the call site | Always call `.Forget()` at the call site, never leave the returned `UniTaskVoid` unobserved |
| Mixed `Awaitable` and UniTask in one workflow | Confusing compile errors, or two different cancellation tokens that don't talk to each other | Pick one per workflow; bridge at the boundary with `.AsUniTask()`/`.AsTask()` if genuinely needed |
| Captured a `CancellationToken` once, didn't thread it through every `await` | Cancelling the token doesn't actually stop the later `await`s in the method | Pass the token to every `await` that accepts one, not just the first |
| `async void` instead of `async UniTaskVoid`/`async UniTask` | Exceptions can't be observed by the caller at all | Use `UniTaskVoid` for fire-and-forget, `UniTask`/`UniTask<T>` for anything awaited |
| `WhenAll` used expecting fail-fast | All tasks run to completion even after one fails | Race a shared `CancellationTokenSource` alongside `WhenAll` if failing fast matters |
| Manual delay loop accumulating `Time.deltaTime` | Reinvents `UniTask.Delay`, usually gets the unscaled-time case wrong | `UniTask.Delay(..., DelayType.UnscaledDeltaTime, ...)` |
| `WaitUntil(() => ...)` in a hot/frequently-awaited path | A closure allocated every call, polled every tick until satisfied | Use the `state` overload with a `static` lambda — see [Closure-free overloads](#closure-free-overloads) |

---

## Review checklist

| Check | Look for |
|---|---|
| Naming | Async method using `Async` suffix instead of `Task` prefix |
| Naming | A genuine coroutine not using the `Routine` suffix |
| Cancellation | A `MonoBehaviour`'s async method with no cancellation token at all |
| Cancellation | A token accepted as a parameter but not passed to every `await` |
| Error handling | An async method body with no `try/catch` |
| Error handling | `OperationCanceledException` caught and swallowed instead of re-thrown |
| Fire-and-forget | A `UniTaskVoid` call with no `.Forget()` |
| Fire-and-forget | `async void` used instead of `UniTaskVoid` |
| Perf | An `async` method that usually completes synchronously, still marked `async` |
| Perf | `WaitUntil`/`WaitWhile` with a capturing (non-`static`) lambda in a frequently-awaited path |
| Mixing | `Awaitable` and UniTask awaited inside the same operation |

---

## Learn more

- [Cysharp/UniTask, GitHub](https://github.com/Cysharp/UniTask) — the source repo; full API reference,
  installation, and the cases this guide doesn't cover (WebGL specifics, IL2CPP considerations,
  `MessagePipe`/`R3` integration).
