# Console & PC Optimization (Unity 2022 LTS)

Platform deltas for **Windows PC, Xbox, PlayStation**, and similar. Read [shared.md](shared.md) first, then apply this file.

Source: [Optimize your console & PC game performance](https://unity.com/resources/performance-optimization-console-pc-games-2022-lts-e-book).

---

## Console/PC-first principles

1. Higher GPU headroom than mobile, but **draw-call and bandwidth costs still matter**.
2. Use **frame time (ms)**; ignore vanity FPS above target.
3. Escalate to **native GPU/CPU tools** (PIX, platform profilers, RenderDoc, Nsight, Radeon tools) once Unity Profiler points at a subsystem.
4. Commit early to a **render pipeline** (URP vs HDRP vs Built-in) and a **rendering path**; don’t mix strategies late.
5. IL2CPP for shipping; optimize IL2CPP flags on consoles for final/profile builds vs fast iteration.

---

## Profiling extras

| Marker | Hint |
|--------|------|
| `Gfx.WaitForPresentOnGfxThread` | Often GPU-bound or render-thread wait |
| `Gfx.PresentFrame` | GPU or VSync |
| `Camera.Render` heavy on render thread | CPU preparing too much for GPU |

- Deep Profiling Support in Build Settings: rare; prefer Call Stacks.
- **Project Auditor** + **Build Report Inspector** for static/build size passes.
- **Heap Explorer** (third-party) optional alongside Memory Profiler for duplicate assets across bundles.

### Native tool map

| Platform | Tools |
|----------|--------|
| Windows / Xbox | PIX, VS Graphics Diagnostics |
| PlayStation | Platform CPU/GPU profilers (registered) |
| NVIDIA | Nsight |
| AMD | μProf, Radeon Developer Tool Suite |
| Intel | VTune, GPA |
| Cross | RenderDoc, Superluminal |

---

## Programming extras

### Update Manager

Thousands of MonoBehaviours with conditional `Update` → custom manager, subscribe only when active (cuts interop).

### Jobs + Burst

Use when CPU-bound on parallelizable blittable work:

- Struct jobs: `IJob` / `IJobParallelFor`
- Blittable data only; results via `NativeContainer` (`NativeArray`, etc.)
- `[BurstCompile]`; prefer `Unity.Mathematics` over `Mathf`
- No managed refs / problematic statics in Burst jobs

### Other

- Disable Stack Trace logging in release when possible.
- Avoid allocating lambdas/closures in hot paths.
- `Transform.SetPositionAndRotation`; pool aggressively.

---

## Project configuration

| Setting | Action |
|---------|--------|
| Scripting backend | **IL2CPP** for release / console |
| PlayStation IL2CPP | Fast options while iterating; Optimized Compile / Remove Unused Code / Optimized link for profile & ship |
| Auto Graphics API | Off; strip unused |
| Quality levels | Only ship tiers |
| Large hierarchies | Flatten |

---

## Assets (PC/console)

- Compression: **BC7** (high) / **DXT1** (lower) on PC/Xbox/PS-class.
- Atlas textures; enforce import via Presets + AssetPostprocessor.
- **Mipmap Streaming** (Quality Settings + texture Advanced) to cut GPU memory.
- Tune **async upload buffer** to largest texture size if uploads stall; don’t over-allocate (memory sticky).
- Addressables for content topology / DLC.
- Polygon density / microtriangles matter even on strong GPUs — LOD distant assets.

---

## Graphics

### Pipeline choice

- **URP:** scalable, Forward / Forward+ / Deferred options; strong for multi-platform.
- **HDRP:** high-end PC/console; use built-in + custom passes; Material/Renderer Priority for sort/overdraw control.
- Strip **shader variants**; remove unused built-in shader settings; optimize Shader Graph (precision, keywords).
- Particles: Particle System vs **VFX Graph** by scale/platform.
- Anti-aliasing: choose technique appropriate to pipeline/path (MSAA vs TAA, etc.) and profile.

### Lighting (shared patterns, higher budget)

- Bake lightmaps; minimize Reflection Probes; disable unneeded shadows; shader fakes where cheaper.
- Light Layers; Light Probes for movers / small props.

### GPU optimization focus

| Topic | Action |
|-------|--------|
| Fill rate / overdraw | Opaque front-to-back; cut transparent stacks; Overdraw / TransparencyOverdraw views |
| Render queues | Understand Built-in vs HDRP priority sorting |
| Batch count | SRP Batcher, instancing, occlusion, fewer unique meshes |
| Frame Debugger | Find accidental draws; confirm batching |
| Post-processing | Profile; avoid PC Asset Store stacks untested on console |

### Console-specific GPU

| Tip | Notes |
|-----|------|
| Enable **Graphics Jobs** | Spread render work across cores (Player Settings) |
| Avoid **tessellation** | Expensive; rare justified cases only |
| Prefer **compute** over geometry shaders | Geometry/vertex can run in depth + shadow passes |
| Watch **wavefront occupancy** | PIX/Razor; vertex-heavy with little pixel work = underuse |
| Shrink shadow map RTs | HDRP HQ defaults (e.g. 4K maps) — reduce and re-profile |
| **Async Compute** | Fill GPU bubbles (e.g. during depth-only shadow pass) with compute |
| HDRP custom/built-in passes | Inject work at defined points |
| Dynamic resolution | Recover GPU frame time under load |
| Multiple cameras | Expensive; prefer URP RenderObjects / HDRP CustomPassVolumes over full extra cameras when possible |

### Culling

- Frustum automatic; Occlusion baked — profile CPU tradeoff.
- `Camera.layerCullDistances` to cull small props earlier per layer.

---

## UI / Audio / Physics / Animation

Same as [shared.md](shared.md), with more CPU/GPU budget — still:

- Split Canvases; UI Toolkit where suitable.
- Physics: CookingOptions, `Physics.BakeMesh`, Box Pruning, solver iterations, non-alloc + batched queries.
- Animation: Generic over Humanoid; update when visible; avoid scale curves; don’t Animator-tween UI.

---

## Console/PC audit extras

```
[ ] Pipeline + rendering path locked; variants stripped
[ ] IL2CPP shipping path; console optimization level correct for profile builds
[ ] Graphics Jobs evaluated on console
[ ] Native GPU capture on a heavy frame (PIX / platform tool)
[ ] Overdraw / transparency heat visualized
[ ] Shadow map resolution justified
[ ] Jobs/Burst considered for CPU-bound systems
[ ] Mipmap Streaming / Addressables plan for memory
[ ] Camera count minimized (RenderObjects/CustomPass before extra cameras)
[ ] Post stack profiled on target console SKU
```
