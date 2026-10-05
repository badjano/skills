---
name: unity-optimization
description: >
  Unity performance optimization playbook from the official 2022 LTS e-books
  (mobile + console/PC), plus Unity 6 Render Graph / Awaitable / DOTS pointers.
  Profile-first workflow, frame budgets, GC/memory, scripting, assets, GPU
  batching, UI, physics, audio, and animation. Use when optimizing Unity
  FPS/frame time, profiling CPU/GPU bottlenecks, reducing GC, draw calls,
  overdraw, build size, or when the user mentions Unity Profiler, mobile
  thermal throttling, URP/HDRP performance, or console/PC optimization.
---

# Unity Optimization (2022 LTS E-Books + Unity 6)

Agent playbook distilled from Unity’s official guides:

- [Optimize your mobile game performance (Unity 2022 LTS)](https://unity.com/resources/optimize-mobile-game-performance-unity-2022lts)
- [Optimize your console & PC game performance (Unity 2022 LTS)](https://unity.com/resources/performance-optimization-console-pc-games-2022-lts-e-book)

**Complements:** `unity-architecture`, `unity-render-graph`, `unity-awaitable`, `unity-dots`, `unity-cli`, `unity-mcp-skill`, `code-architecture`.

### Unity 6 performance stacks (delegate, don’t reinvent here)

| Bottleneck class | Prefer | Skill |
|------------------|--------|--------|
| Custom URP/HDRP pass VRAM / barriers | Declarative Render Graph (transient aliasing, pass cull) | `unity-render-graph` |
| Async GC / coroutine overhead | `UnityEngine.Awaitable` (pooled; single-await) | `unity-awaitable` |
| Thousands of similar agents (CPU cache) | ECS chunks + Burst `IJobEntity` | `unity-dots` |
| Editor UI CPU (IMGUI OnGUI spam) | UI Toolkit retained tree | `unity-editor-extensibility` |

---

## How to use

1. Detect platform target → read only the matching reference file(s).
2. **Never guess bottlenecks.** Profile → change one thing → compare.
3. Prefer frame time (ms) over FPS as the primary metric.
4. Balance gains vs complexity; many tips add maintenance cost.

### Routing

| Task | Read |
|------|------|
| Any optimization / unknown bottleneck | [shared.md](shared.md) first |
| iOS / Android / handheld thermal / battery | [mobile.md](mobile.md) |
| PlayStation / Xbox / Windows PC / Switch PC-class | [console-pc.md](console-pc.md) |
| Source URLs / edition notes | [sources.md](sources.md) |

---

## Mandatory workflow

```
[ ] 1. Set target FPS → compute frame budget (ms)
[ ] 2. Profile Development Build on target hardware (min-spec + max-spec)
[ ] 3. Decide CPU-bound vs GPU-bound (Profiler Timeline + Gfx.* markers)
[ ] 4. Save Profiler .data BEFORE changes
[ ] 5. Fix the hottest marker / biggest GC.Alloc / worst batch cost only
[ ] 6. Profile again → Profile Analyzer Compare view
[ ] 7. Repeat until inside budget (with thermal headroom on mobile)
```

### Frame budgets

| Target | Hard budget | Mobile working budget (~65%) |
|--------|-------------|------------------------------|
| 30 fps | 33.33 ms    | ~22 ms                       |
| 60 fps | 16.66 ms    | ~11 ms                       |

- **PC/console:** stay within hard budget.
- **Mobile:** plan ~65% of budget so the SoC can cool; short spikes OK, sustained over-budget causes thermal throttle.

### CPU vs GPU bound (quick read)

| Marker / signal | Likely bound |
|-----------------|--------------|
| `Gfx.WaitForCommands` | Main-thread / CPU bottleneck |
| `Gfx.WaitForPresent` / `Gfx.WaitForPresentOnGfxThread` / `Gfx.PresentFrame` | GPU or VSync wait |
| Render thread busy in `Camera.Render` | CPU sending too much to GPU |
| High batches / SetPass, low GPU time | CPU draw-call bound |

---

## Hard rules (both platforms)

1. **Profile on device**, not only in Editor. Development Build + Profiler connect.
2. **Measure in ms**, not FPS percentages. A 900→450 fps “halving” is only ~1.1 ms.
3. **No GC.Alloc in hot gameplay loops.** Prefer Incremental GC as a mitigator; eliminate allocs as the real fix.
4. **No `Debug.Log` / string work in `Update`/`FixedUpdate`/`LateUpdate` in player builds.** Gate with `[Conditional("ENABLE_LOG")]`.
5. **Cache components** in `Awake`/`Start`; never `GetComponent` / `Find` / `Camera.main` (pre-2020.2) per frame.
6. **Pool** instead of `Instantiate`/`Destroy` spam.
7. **`Renderer.sharedMaterial`** for reads that must not break batches; `Renderer.material` clones and breaks batching.
8. **Property IDs:** `Animator.StringToHash` / `Shader.PropertyToID` — cache once.
9. **Empty `Update`/`LateUpdate` is still expensive** — delete or `#if UNITY_EDITOR`.
10. **Textures:** POT, compress for platform, disable Read/Write, atlas when possible, disable unused mipmaps (UI/2D).
11. **Meshes:** disable Read/Write, unused rigs/blendshapes/normals/tangents; reduce density (microtriangles hurt).
12. **UI:** split static vs dynamic Canvases; disable invisible UI; limit Raycast Targets; avoid nested Layout Groups.
13. **Physics:** primitive colliders > mesh; move Rigidbodies with physics APIs; disable Auto Sync Transforms; enable Reuse Collision Callbacks; non-alloc queries.
14. **One camera when possible.** Each camera has real CPU cost (worse on mobile).

---

## Output format (when advising or auditing)

```markdown
## Verdict
CPU-bound | GPU-bound | Memory/GC | Mixed — [one sentence]

## Evidence
- Tool: Profiler / Memory Profiler / Frame Debugger / native tool
- Marker or metric: …
- Frame time: X ms (budget Y ms)

## Fix (ordered)
1. Highest-impact change …
2. …

## Verify
Profile again on [device]; Compare .data before/after.
```

---

## Anti-patterns

| Don't | Do |
|-------|----|
| Optimize from hunches | Profile → fix hottest cost |
| Chase FPS % | Chase ms inside budget |
| Deep Profile by default | Call Stacks on `GC.Alloc` first |
| Leave Incremental GC as the only GC strategy | Remove hot-path allocations |
| One giant Canvas | Static / dynamic Canvas split |
| Dynamic lights + realtime shadows everywhere | Bake + Light Probes + blobs |
| `AddComponent` at runtime | Prefab with components ready |
| Nested Layout Groups | Anchors or one-shot layout then disable |

---

## Expansion

Keep [SKILL.md](SKILL.md) as the router. Put dense tip lists in reference files. Prefer Unity 2022 LTS guidance for platform tips; for Unity 6 Render Graph / Awaitable / DOTS implementation details, defer to the specialized skills above. Note Unity 6 successor e-books in [sources.md](sources.md) when relevant.
