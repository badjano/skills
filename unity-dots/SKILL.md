---
name: unity-dots
description: >-
  Unity DOTS/ECS guidance: archetypal 16KB chunks, structural changes, shared
  components, IJobEntity/Burst, enabled-bit filtering, and when to use ECS vs
  MonoBehaviour. Use when writing Entities, Components, Systems, IJobEntity,
  Baker/authoring, structural changes, or optimizing large agent simulations.
---

# Unity DOTS / ECS

Source paradigm: Unity Editor Deep Research — cache-coherent archetypal memory + Burst jobs. Complements `unity-architecture` (Jobs/Burst basics) and `unity-optimization`.

**Use ECS when** thousands of similar agents/sim elements need parallel, cache-friendly updates. **Keep MonoBehaviour** for sparse gameplay objects, UI shells, and editor-facing authoring unless the project is ECS-first.

---

## Memory model (non-negotiable mental model)

- An entity’s **archetype** = exact set of component types.
- Same-archetype entities live in **16KB chunks** as contiguous value-type columns.
- Iteration is linear arrays → prefetch-friendly; avoid random managed references in hot components.
- **SharedComponentData**: one value per chunk (e.g. shared mesh/material). Changing it on one entity = **structural change** (move to another chunk).

### Structural changes are expensive

Creating/destroying entities, adding/removing components, or changing shared components:

- Stalls job dependency chains
- Copies memory between chunks

**Batch** structural changes; prefer enableable components / `IEnableableComponent` + bitmask filtering over add/remove spam.

Destroy packing: last entity in chunk swaps into freed slot — keep logic order-independent where possible.

---

## Systems & jobs

- Prefer **`IJobEntity`** (or idiomatic foreach schedules) with automatic read/write dependency sync.
- Schedulers skip sync when component types have **no matching entities** — keep queries tight.
- Burst: blittable data only; `Unity.Mathematics`; no managed `UnityEngine.Object` inside jobs.
- Use `ArchetypeChunk.GetEnabledMask()` / SIMD range helpers to skip disabled entities — don’t bool-check per entity in Burst hot loops when masks exist.

### Hierarchy / transforms

`LocalToWorld` and child buffers are hot paths — minimize unnecessary parent churn; note Entities 1.x reduced internal capacities on `LinkedEntityGroup` / `Child` to improve chunk packing (empty buffer overhead matters inside 16KB).

---

## Authoring bridge

- Author in subscenes with **Bakers** → burst-friendly runtime components.
- Do not mutate authoring MonoBehaviours at runtime as if they were ECS state.
- Config that must stay designer-friendly can remain SO/MB on the authoring side; bake into components.

---

## Anti-patterns

| Don't | Do |
|-------|-----|
| Per-frame Add/Remove component | Enableable components or state flags |
| Managed class components in hot paths | `IComponentData` structs |
| Main-thread `foreach` over all entities for heavy sim | `IJobEntity` + Burst Schedule |
| Touch `UnityEngine.Object` inside Burst | Bake IDs/handles; resolve on main if needed |
| One-off gameplay object forced into ECS | MonoBehaviour + Jobs if batch work only |

---

## Agent checklist

- [ ] Hot sim data is `IComponentData` in chunks, not scattered MB fields
- [ ] Structural changes batched / rare
- [ ] Jobs Burst-safe; deps declared via query attributes
- [ ] Shared components used only when truly chunk-constant
- [ ] Verified with Entities package version awareness (1.0–1.5+ behaviors)

## Related

- MonoBehaviour standards: `unity-architecture`
- Profiling: `unity-optimization`
- Async outside ECS: `unity-awaitable`
