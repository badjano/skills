---
name: unity-awaitable
description: >-
  UnityEngine.Awaitable async rules for Unity 6: PlayerLoop yields, main/background
  thread marshaling, single-await pooling limits, CancellationToken across domain
  reload, and when to use Task/UniTask instead. Use when writing async Unity code,
  replacing coroutines, BackgroundThreadAsync, NextFrameAsync, or fixing
  OperationCanceledException after domain reload / play mode changes.
---

# UnityEngine.Awaitable (Unity 6)

Source paradigm: Unity Editor Deep Research — pooled, PlayerLoop-native async. Complements `unity-architecture`.

| Prefer | When |
|--------|------|
| **`UnityEngine.Awaitable`** | Unity 6+ engine waits, frame sync, simple background→main marshaling |
| **UniTask** | Project already standardized on it, or need multi-await / richer extras |
| **`Task`** | Multiple concurrent awaiters on one operation; interop with .NET libs |
| **Coroutine** | Legacy only — no return values, weak exceptions, GO-lifecycle coupling |

---

## PlayerLoop yields

```csharp
await Awaitable.NextFrameAsync();
await Awaitable.WaitForSecondsAsync(0.5f, destroyCancellationToken);
await Awaitable.FixedUpdateAsync();
await Awaitable.EndOfFrameAsync();
```

Thread hop:

```csharp
await Awaitable.BackgroundThreadAsync();
// CPU work — no UnityEngine.Object touch
await Awaitable.MainThreadAsync();
// Safe to touch Unity API again
```

Continuations run **synchronously** on completion (same frame/phase) — unlike ThreadPool `Task` resumes. Use that for frame-deterministic game state.

---

## Hard constraints (agents must not violate)

### 1. Single-await only

Awaitable instances are **pooled on completion**. Awaiting the same instance twice → undefined behavior / asserts / deadlock.

- One consumer → `Awaitable` is fine.
- Many consumers → return / expose `Task` (or UniTask), not a shared Awaitable.

### 2. CancellationToken is mandatory for robust code

Combine `BackgroundThreadAsync` + .NET waits (`Task.Delay`, etc.) carefully. After **domain reload** or Play Mode teardown, captured contexts can invalidate → `OperationCanceledException` via engine generation checks (`CodeLoadedScope`).

Always:

- Pass `destroyCancellationToken` / `GetCancellationTokenOnDestroy()` / explicit `CancellationToken`
- Treat cancel as normal shutdown, not a mystery bug
- Do not let background work touch Unity objects after cancel

### 3. No Unity API on background thread

After `BackgroundThreadAsync()`, only pure CPU / blittable data. Marshal back with `MainThreadAsync()` before any `UnityEngine.Object` use.

### 4. Do not replace multi-await UniTask fan-out with Awaitable

If existing code `UniTask.WhenAll` / multiple awaiters on one handle — keep UniTask/Task.

---

## Pattern

```csharp
async Awaitable LoadAsync(CancellationToken ct)
{
    var data = await FetchOnBackground(ct);
    await Awaitable.MainThreadAsync();
    Apply(data); // Unity API
}

async Awaitable<byte[]> FetchOnBackground(CancellationToken ct)
{
    await Awaitable.BackgroundThreadAsync();
    ct.ThrowIfCancellationRequested();
    return ComputeOrIO(ct);
}
```

---

## Agent checklist

- [ ] New Unity 6 async uses Awaitable unless project is UniTask-first or needs multi-await
- [ ] Never await one Awaitable instance twice; never share it to multiple callers
- [ ] CancellationToken on all waits that can outlive a GO / domain
- [ ] Background work does not touch Unity objects
- [ ] No coroutine for new code with return values / error propagation

## Related

- Broader Unity standards: `unity-architecture`
- EditMode/PlayMode tests with async: `unity-test-framework`
