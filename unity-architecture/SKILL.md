---
name: unity-architecture
description: >
  Unity-specific architecture, editor workflow, performance optimization,
  and Unity systems. Activate when writing/modifying Unity code:
  MonoBehaviours, ScriptableObjects, physics, networking, pooling, VR
  performance, or editor tooling. Unity 6 defaults: Awaitable async, UI Toolkit
  editor UI, Render Graph passes — see specialized unity-* skills.
---

# Unity Architecture & Performance

This skill covers Unity-specific standards. For general SOLID, service architecture, event patterns, and readability rules, see `code-architecture`.

### Unity 6 skill router

| Concern | Skill |
|---------|--------|
| Custom inspectors, SerializedObject, UI Toolkit, Overlays | `unity-editor-extensibility` |
| URP/HDRP custom passes / Render Graph | `unity-render-graph` |
| `UnityEngine.Awaitable` / async rules | `unity-awaitable` |
| ECS / DOTS / IJobEntity | `unity-dots` |
| EditMode/PlayMode tests | `unity-test-framework` |
| uGUI Canvas / HUD | `unity-canvas-ui-expert` |
| Profile-first FPS / GC / GPU | `unity-optimization` |

Unity 6 favors **declarative, dependency-declared systems** (UI Toolkit trees, Render Graph topology, ECS chunks) over immediate-mode loops and ad-hoc memory. Prefer those stacks for new work; keep legacy paths only when the project already depends on them.

---

## 0. Editor-Time Setup Over Runtime Checks

**Do the work in the editor, not at runtime.** If something can be configured, wired, validated, or caught at edit-time, it must be — never defer to runtime what the editor can guarantee.

This means:
- **Wire references in prefabs/scenes**, not via `GetComponent`/`Find*` at runtime
- **Validate configuration in `OnValidate()` and editor scripts**, not with runtime null-checks or try/catch
- **Set up component state in the inspector or `Reset()`**, not in `Awake()`/`Start()` init blocks
- **Catch errors at import/save time** with editor tools and custom inspectors, not during gameplay
- **Use `[RequireComponent]`** to enforce component dependencies at the prefab level
- **Build editor tooling** (custom windows, auto-wirers, validators) to eliminate manual setup errors before they become runtime bugs

Runtime initialization should only handle things that genuinely cannot be known until play — network state, dynamic spawns, player input. Everything else is an editor responsibility.

### Tooling preference (every Unity game)

**Prefer Editor tools over shell scripts.** When accelerating setup, validation, prefab generation, or one-click fixes:

1. Add a `[MenuItem("Tools/...")]` (and optional `InitializeOnLoad` auto-run once) under an `Editor/` folder.
2. Mutate assets via **`SerializedObject` / `SerializedProperty`** (Undo, Prefab overrides, multi-select) — see `unity-editor-extensibility`. New inspector UI → **UI Toolkit** (`CreateInspectorGUI`), not IMGUI.
3. If headless/CI must call the same logic, expose a static `Cli*`/`executeMethod` twin on that Editor class — do **not** add a project `.ps1`/`.bat`/`.sh` wrapper.
4. Only invoke raw `Unity.exe -batchmode` when the Editor GUI is closed or the user explicitly asks for CLI compile; still call Editor methods via `-executeMethod`, never a repo script.

Do not introduce `Tools/*.ps1` (or similar) for Unity workflows unless the user explicitly requests a shell script.

### Mandatory verification (Unity MCP + CLI)

**Do not claim a Unity change works until the target project was verified in-editor or via CLI.** Code edits, DLL drops, package upgrades, and Unity-version migrations all require this gate.

| Priority | When | How |
|----------|------|-----|
| 1 | Target project Editor is open **and** Unity MCP is connected to **that** project | Use **unity-mcp** / **unity-mcp-skill**: confirm `Application.dataPath`, `scriptCompilationFailed == false`, console errors cleared, load/check critical types or run a smoke MenuItem |
| 2 | MCP points at the wrong project, or Editor is closed | Use **unity-cli**: `Unity.exe -batchmode -nographics -quit -accept-apiupdate -projectPath … -logFile Temp\cli-compile.log`, then parse `error CS` / exit code |
| 3 | Same project is open in Editor (mutex) and MCP is wrong/unavailable | Say so explicitly; ask user to focus MCP on the target project **or** close the Editor so CLI can run. Do **not** invent a “probably fine” verdict |

Always confirm MCP project identity first (`Application.dataPath` / product name). Multiple Unity instances are common — wrong-project MCP results are not verification.

Also follow **unity-mcp-skill** (console after compile) and **unity-cli** (log parse recipe).

---

## 1. SOLID in Unity

The SOLID principles (defined in `code-architecture`) have specific Unity applications:

### S — Single Responsibility
- A MonoBehaviour handles **one concern**: visuals OR input OR physics OR networking — not all of them.
- If `Update()` has more than 3 distinct logic blocks, split into separate components.

### O — Open/Closed
- Use **ScriptableObject-driven config** to change behavior without editing code. New weapon? New SO asset, not a new `if` branch.
- The catalog-item module pattern (see section 8) is the preferred example of composition over inheritance.

### L — Liskov Substitution
- If a base catalog SO exposes `GetPrice()`, every subclass must return a valid price.
- Photon callbacks (`IPunObservable`, `IOnEventCallback`) must fully implement their contracts. A partial implementation causes silent networking bugs.

### D — Dependency Inversion
- MonoBehaviours receive dependencies via **SerializeField** (editor-injected) or **interface lookup** — never by finding concrete types with `FindObjectOfType<ConcreteClass>()`.
- Services live on `DontDestroyOnLoad` GameObjects or use `ServiceLocator`.

---

## 2. SerializeField Conventions

### Prefer SerializeField over GetComponent — always

**Do not use `GetComponent` at runtime.** All component references should be resolved at edit-time and stored in `[SerializeField]` fields. This is faster (zero runtime cost), explicit (you see what's wired in the inspector), and catches missing references before play mode.

Use one of these approaches to populate SerializeField references:

1. **`Reset()` method** — auto-populates when the component is added in the editor:

```csharp
private void Reset()
{
    _rigidbody = GetComponent<Rigidbody>();
    _collider = GetComponentInChildren<Collider>();
    _audioSource = GetComponentInChildren<AudioSource>();
}
```

2. **Editor scripts / custom inspectors** — for complex wiring, cross-prefab references, or batch operations. Use `OnValidate()` or dedicated editor tools (e.g., `ItemComponentAutoWirer`) to auto-resolve references.

3. **Manual inspector assignment** — acceptable for scene-level or cross-hierarchy references that can't be auto-resolved.

**Rules:**
- Never call `GetComponent`, `GetComponentInChildren`, or `GetComponentInParent` in `Awake()`, `Start()`, `OnEnable()`, or any runtime method. Wire it in the editor instead.
- The only exception is dynamically spawned objects where the reference truly cannot be known at edit-time (e.g., runtime-instantiated prefabs wiring to scene objects). Even then, prefer passing the reference via an init method rather than searching for it.
- If you need a reference that `Reset()` can't find (different hierarchy, scene object), write an editor script or use `OnValidate()` to populate it.

### Never null-check a SerializeField

A `[SerializeField]` field is a contract: it **must** be assigned in the inspector or via `Reset()`. Do not write `if (myField == null)` guards around them — if they're null, that's a configuration bug that should surface immediately, not be silently swallowed.

Exceptions:
- Fields marked with `[CanBeNull]`, `[Optional]`, or a similar attribute — these explicitly signal the field may be empty
- Fields on components that are reused across prefabs where the reference is genuinely optional by design

If a reference is truly optional, mark it clearly so the intent is obvious:

```csharp
[SerializeField, Tooltip("Optional — leave empty to disable trail")]
private TrailRenderer _optionalTrail;
```

---

## 3. Zero-Allocation Hot Paths (Mandatory)

Mandatory for any code in `Update`, `FixedUpdate`, or high-frequency callbacks. VR requires 72-90hz; even minor GC pauses cause motion sickness.

### Physics Queries
Always use non-allocating variants with pre-allocated buffers and strict LayerMasks:

```csharp
private static readonly RaycastHit[] HitBuffer = new RaycastHit[16];

int count = Physics.RaycastNonAlloc(origin, direction, HitBuffer, maxDistance, layerMask);
for (int i = 0; i < count; i++) { /* process HitBuffer[i] */ }
```

### Strings & Collections
- **No `+` or `$""` inside `Update()`:** Use `FixedString32Bytes` or a pooled `StringBuilder`
- **Collections:** No LINQ (`.Where`, `.Select`, `.ToList`) in hot paths — use `for` loops
- **Boxing:** Don't pass value types as `object` in hot paths

### General
- All component references must be `[SerializeField]` wired at edit-time (see section 2) — no `GetComponent` at runtime
- Cache `Camera.main` in `Awake`/`Start` — it calls `FindObjectOfType` internally
- Use `CompareTag("Tag")` instead of `gameObject.tag == "Tag"` (avoids string allocation)

---

## 4. Object Pooling (Project-Specific)

Never raw `Instantiate`/`Destroy` for frequently spawned objects. Use the project's pool system.

### Key classes

| Class | Role |
|---|---|
| `GenericPool<T>` | Queue-based generic pool. `Instantiate()`, `Destroy(T)`, `Populate(int)` |
| `NetworkedObjectPool` | Singleton, implements `IPunPrefabPool`. Handles networked + local spawning |
| `LocalObjectPool` | Singleton for non-networked objects only |
| `PooledObject` | MonoBehaviour on every pooled prefab. Lifecycle, destroy timer, auto-detects networked vs local |
| `NetworkSpawnEntryContainer` | ScriptableObject at `Resources/Pool/`. Configures all pool entries by category |
| `SpawnEntry` | Serializable config: prefab ref, `poolable` flag, `startingCount`, `networked` flag |

### Spawn/despawn API

```csharp
// Networked spawn (auto-routes through Photon if in room)
var go = NetworkedObjectPool.InstantiateObject(prefabId, position, rotation);

// Local-only spawn
var go = LocalObjectPool.InstantiateObject(prefabId, position, rotation);

// Return to pool (works for both networked and local)
pooledObject.ReturnToPool();
```

### Rules
- Every pooled prefab must have a `PooledObject` component
- Register prefabs in `NetworkSpawnEntryContainer` SO (15 categories: particles, cosmetics, playerItems, weapons, projectiles, etc.)
- `PooledObject` auto-detects networked mode via `PhotonView` presence — no manual branching needed
- Non-owner destruction requests go through RPC to the owner
- `PlayerItemRegistry.SpawnItemAsGameObject()` routes through `NetworkedObjectPool` internally
- Pool queues grow on demand — no capacity limit, but set `startingCount` to avoid runtime alloc spikes

---

## 5. Networking (Photon PUN2)

- Only the owner (`photonView.IsMine`) processes input and local state
- Critical state (health, scores, inventory) validated by Master Client
- RPCs: use `Rpc_` prefix, pass IDs/indices not strings, throttle unchanged data
- Use `OnPhotonSerializeView` with change detection — don't sync every frame

---

## 6. Async Patterns (Awaitable / UniTask)

Full rules: **`unity-awaitable`**. Summary:

| Stack | Use when |
|-------|----------|
| **`UnityEngine.Awaitable`** | Unity 6 default for new engine async (frame waits, background↔main) |
| **UniTask** | Project already UniTask-first, or multi-await / `WhenAll` fan-out |
| **`Task`** | Multiple awaiters on one operation; .NET interop |
| **Coroutine** | Legacy only |

```csharp
// Unity 6 preferred
async Awaitable DoWorkAsync(CancellationToken ct)
{
    await Awaitable.WaitForSecondsAsync(0.5f, ct);
}

// UniTask — OK when project standard or multi-consumer
async UniTask DoWorkUniTaskAsync(CancellationToken ct)
{
    await UniTask.Delay(500, cancellationToken: ct);
}
```

**Never await the same `Awaitable` twice** (pooled on completion). Always pass `destroyCancellationToken` / `GetCancellationTokenOnDestroy()` / explicit `CancellationToken` — domain reload + background work cancels aggressively.

### Centralized Update Management
Instead of 1,000 `MonoBehaviour.Update()` calls, prefer a centralized `TickManager` that iterates over an array of `ITickable` objects when applicable.

---

## 7. Job System, Burst & DOTS

Use C# Jobs + Burst for CPU-bound batch operations (damage falloff, spatial queries, AI evaluation) when applicable.

- Always `Dispose()` `NativeArray` / `NativeList` — prefer explicit cleanup in `finally` blocks
- Use `[ReadOnly]` on all input arrays to enable Burst load-store optimizations
- Never access managed types (classes, strings, `UnityEngine.Object`) inside Burst jobs
- Use `math.*` (`Unity.Mathematics`) instead of `Mathf.*` inside jobs for SIMD instructions

For large homogeneous simulations (thousands of agents), prefer **ECS/DOTS** (`unity-dots`): archetypal 16KB chunks, minimize structural changes, `IJobEntity` + enableable components over Add/Remove spam.

---

## 8. Data Architecture & ScriptableObject Hierarchy

### General rule
- **Config** = `ScriptableObject` (read-only at runtime)
- **State** = plain C# struct or class (mutable at runtime)
- Never mutate SO fields at runtime — copy to runtime state

### Catalog item pattern (composition over inheritance)

Prefer a base catalog SO with **serialized modules** rather than a deep type tree:

```
CatalogItemData (base SO)
├── WeaponItemData
├── CosmeticItemData
├── BundleItemData
└── CharacterItemData
```

### Composition modules (illustrative)

| Module | Purpose |
|---|---|
| `CurrencyPriceModule` | Soft/hard currency prices |
| `SessionCurrencyModule` | In-session / match currency prices |
| `SettingsModule` | Consumable / stackable flags |
| `ShopConfigModule` | Shop availability flags |
| `IAPModule` | Real-money IAP (SKU, store price) |
| `ConsumableModule` | Remaining uses tracking |

### Container nesting

Organize catalog assets in nested ScriptableObject containers (groups → subgroups → item refs). At runtime, flatten into a `Dictionary<string, CatalogItemData>` (or similar) keyed by stable item id, and sync with your backend catalog/inventory if you have one.

Keep enums for item type, purchase channel, rarity, and category project-specific — define them in your game’s data layer, not as hardcoded skill assumptions.

---

## 9. Unity Logging

Use `Debug.Log` / `Debug.LogWarning` / `Debug.LogError` — see `code-architecture` for general logging discipline. Unity-specific additions:

- `LogError` for broken invariants, null required refs
- `LogWarning` for degraded-but-functional states
- Never log in `Update`, `FixedUpdate`, or per-frame callbacks

---

## Quick Checklist

Before submitting any Unity code change:
- [ ] **Verified via Unity MCP (correct project) or unity-cli compile** — not assumed
- [ ] No allocations in Update/FixedUpdate
- [ ] Physics uses NonAlloc with static buffers
- [ ] No runtime GetComponent — all refs are SerializeField wired at edit-time
- [ ] Setup/validation done in editor (`Reset()`, `OnValidate()`, editor tools) — not at runtime
- [ ] Camera.main cached
- [ ] Async uses Awaitable (Unity 6) or project UniTask standard — always with cancellation; no double-await Awaitable
- [ ] New editor UI via UI Toolkit + SerializedObject (`unity-editor-extensibility`)
- [ ] New URP/HDRP custom passes via Render Graph (`unity-render-graph`)
- [ ] Events unsubscribed in OnDestroy
- [ ] No comments added (unless user explicitly requested them)
- [ ] No unnecessary Debug.Log added
- [ ] SerializeFields auto-wired in `Reset()` where possible
- [ ] No null-checks on SerializeFields (unless marked optional)
- [ ] Frequently spawned objects use pool (`NetworkedObjectPool` / `LocalObjectPool`)
- [ ] ScriptableObject not mutated at runtime
- [ ] CompareTag used for tag checks
- [ ] Assembly boundaries respected — no upward/cyclic dependencies
