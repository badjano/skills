# Shared Optimization Reference

Applies to **mobile and console/PC** unless a platform file overrides it. Distilled from both Unity 2022 LTS optimization e-books.

---

## Profiling

### Tools (start here)

| Tool | Use for |
|------|---------|
| Unity Profiler | CPU, memory, audio, physics, rendering modules |
| Profile Analyzer | Aggregate frames; **Compare** before/after |
| Memory Profiler | Snapshots, leaks, duplicates, fragmentation |
| Frame Debugger | Per-draw construction of a frame; batching issues |
| Game view Stats | Batches, SetPass, tris/verts (directional only — profile builds for truth) |

### Practice

- Profile **early and often**, not only near ship.
- Development Build on **target device**; Autoconnect Profiler only when you need first frames (adds startup cost).
- Default modules: **CPU + Memory**. Add Renderer / Physics / Audio when relevant.
- Save `.data` before changes; compare after.
- Deep Profiling: last resort (high overhead). Prefer **Call Stacks** on `GC.Alloc` / `JobHandle.Complete`.
- Increase frame count (Preferences → Analysis → Profiler) up to 2000 only when needed.

### Native tools (escalate when Unity tools aren’t enough)

- **iOS:** Xcode, Instruments, Apple Frame Capture
- **Android:** Android Studio Profiler, Arm Mobile Studio, Snapdragon Profiler
- **PC:** RenderDoc, PIX, Intel VTune/GPA, NVIDIA Nsight, AMD μProf/Radeon tools, Superluminal
- **Console:** PIX (Xbox), platform CPU/GPU profilers (registered developer portals)
- **WebGL:** Firefox Profiler, Chrome DevTools Performance

### Static / build helpers

- **Project Auditor** (experimental): static script/settings analysis
- **Build Report Inspector**: build time + disk footprint

---

## Memory & GC

Unity: stack for small value locals; managed/native heaps for larger/long-lived data. Boehm GC **stops the world** unless Incremental GC spreads work.

### Kill these alloc sources

| Source | Fix |
|--------|-----|
| Strings / concat in loops | `StringBuilder`; avoid per-frame formatting |
| JSON/XML parse at runtime | ScriptableObjects, MessagePack, Protobuf |
| `gameObject.tag == "..."` | `CompareTag` |
| Boxing value→object | Generics / concrete overloads |
| `new WaitForSeconds` in coroutine | Cache and reuse |
| LINQ / Regex in hot paths | Explicit loops |
| `new List<>` every frame | Member list + `Clear()` |
| Array-returning Unity APIs in loops | Cache results |

### Policy

- **Zero GC.Alloc** in main gameplay loops is the goal.
- `System.GC.Collect()` only when a hitch is hidden (loading, menu, cutscene).
- Enable **Incremental Garbage Collector** to blunt spikes; still remove allocs.
- Memory Profiler: Unity Objects tab for duplicates; All of Memory for full breakdown.

---

## Programming & architecture

### PlayerLoop

Know `Awake` → `OnEnable` → `Start` → `Update` / `FixedUpdate` / `LateUpdate` order. User code appears under **PlayerLoop** in the Profiler.

### Per-frame cost

- Move logic out of `Update`/`LateUpdate`/`FixedUpdate` unless it must run every frame.
- Time-slice: either every *n* frames **or** (better) 1/*n* of the work each frame for stable cost.
- Avoid heavy work in first-scene `Awake`/`Start`/`OnEnable` (extends load / first frame).
- Remove empty Unity message methods.
- Prefer Prefabs over runtime `AddComponent`.
- Cache references; use object pools (`UnityEngine.Pool` / custom).
- `Transform.SetPositionAndRotation` once; `Instantiate(prefab, parent, pos, rot)` to avoid double transforms.
- Shared config → **ScriptableObject** (flyweight), not duplicated MonoBehaviour fields.
- Large projects with thousands of idle `Update` checks → custom **Update Manager** (subscribe/unsubscribe) to cut C++/C# interop.

### Console/PC extras (also in console-pc.md)

- Avoid hot-path lambdas/closures that allocate delegates.
- Jobs + Burst for parallel blittable work (`IJob` / `IJobParallelFor`, `NativeArray`, `[BurstCompile]`, `Unity.Mathematics`).

---

## Project configuration

| Setting | Action |
|---------|--------|
| Auto Graphics API | Disable; keep only APIs you ship |
| Unused Quality levels | Remove |
| Unused CPU architectures | Strip |
| Physics unused | Disable Auto Simulation / Auto Sync Transforms |
| Scripting backend | Prefer **IL2CPP** for release (Mono OK for fast local iteration) |
| Hierarchies | Flatten when parenting isn’t needed (helps multithreaded Transform updates) |
| Stack traces (release) | Disable if logging errors in the wild without needing stacks |

---

## Assets

### Textures

- Lower Max Size to acceptable visual quality.
- Powers of two for mobile compression formats.
- Atlas (Sprite Atlas / TexturePacker / DCC).
- **Read/Write Off** (doubles memory when on).
- Mipmaps off for fixed-size UI/sprites; on for 3D distance-varying.
- Platform compression:
  - Mobile / Switch: **ASTC** (legacy: PVRTC old iOS, ETC2 old Android)
  - PC / Xbox / PS: **BC7** (HQ) or **DXT1** (lower)

### Meshes

- Mesh compression for disk; disable Read/Write; strip unused rigs/blendshapes/normals/tangents.
- Cut unseen faces; prefer normal maps over density; watch **microtriangles**.
- Player Settings: Vertex Compression, Optimize Mesh Data when appropriate.

### Pipeline hygiene

- **Presets** + **AssetPostprocessor** to enforce import rules.
- **Addressables** for async local/remote content and smaller initial builds.
- Unity DataTools for unused assets / dependency analysis.
- Console/PC: Mipmap Streaming, async upload buffer tuning (`QualitySettings.asyncUploadBufferSize`).

---

## Graphics & GPU (shared)

### Batching priority

1. **SRP Batcher** (URP/HDRP) — few compatible shaders, persist material data
2. **GPU Instancing** — many identical mesh+material
3. **Static batching** — non-moving, shared material (more memory)
4. **Dynamic batching** — only tiny meshes (≤300 verts / ≤900 attributes); otherwise leave off

### Rules

- Fewer textures → fewer materials → better batching.
- Large lightmap atlases (watch memory).
- Never touch `Renderer.material` casually.
- Bake lighting; limit realtime lights/shadows; Light Probes for movers.
- Light Layers / culling masks to limit light influence.
- LOD Groups; Occlusion Culling where profile proves benefit.
- Minimize Reflection Probes (resolution, masks, compression).
- SkinnedMeshRenderer only when needed; `BakeMesh` + MeshRenderer when idle.
- Overdraw: transparent stacks, particles, heavy UI, multi-pass shaders — visualize and cut.
- Post-processing: profile; keep mobile art direction conservative.

### URP paths (esp. mobile)

| Path | Notes |
|------|--------|
| Forward | Default mobile; limited realtime lights/object |
| Forward+ | Spatial light culling; more lights (2022 LTS+) |
| Deferred | Many dynamic lights; MSAA no; normals encoding cost on mobile |

---

## UI

### UGUI

- **Split Canvases** by update frequency (static vs dynamic).
- Same Z / materials / textures within a Canvas when possible.
- Disable invisible UI; disable **Canvas component** (not whole GO) when only hiding.
- GraphicRaycaster only on interactive elements; disable unused Raycast Target; disable Ignore Reversed Graphics if unused.
- Avoid Layout Groups for static UI; don’t nest them; disable after setup if one-shot.
- Pool UI rows for large lists/grids.
- Merge stacked overlays to reduce overdraw.
- Fullscreen menus: disable 3D camera / hidden canvases; lower `targetFrameRate`.
- Assign Event/Render Camera explicitly (blank → expensive `Camera.main` path). Prefer Screen Space Overlay when possible.

### UI Toolkit

- Prefer Flexbox layouts over manual positioning.
- No heavy create/manipulate in `Update`.
- Unsubscribe unused events; keep USS lean; profile redraws.

---

## Audio

| Clip size | Load Type |
|-----------|-----------|
| Small (&lt; ~200 KB) | Decompress on Load |
| Medium | Compressed in Memory |
| Large (music) | Streaming |

- Mono / Force To Mono for 3D spatial sources.
- Source assets: uncompressed WAV (avoid double lossy recompress).
- Vorbis (or MP3 non-looping); ADPCM for short frequent SFX.
- Mute = don’t leave silent sources loaded; destroy/unload when appropriate.
- Optimize AudioMixer group count / effects when profiling shows cost.

---

## Animation

- Prefer **Generic** rig over Humanoid unless IK/retargeting required (Humanoid ~30–50% more CPU).
- Don’t use Animator for single UI property tweens — legacy Animation, code tween, or DOTween-style.
- Avoid scale curves when possible; culling mode: update when visible.
- Console/PC: more Animator budget, but same principles apply.

---

## Physics

- Prebake Collision Meshes; simplify Layer Collision Matrix.
- Disable **Auto Sync Transforms**; enable **Reuse Collision Callbacks**.
- Primitive / convex approx &gt; MeshCollider.
- Move with `MovePosition` / `AddForce` in `FixedUpdate`, not Transform teleports.
- Align Fixed Timestep with target frame rate; lower Maximum Allowed Timestep to cap hitch cost.
- Non-alloc ray/overlap queries; batch raycasts where available.
- Physics Debugger to visualize collision matrix issues.
- Console/PC extras: CookingOptions, `Physics.BakeMesh`, Box Pruning for large scenes, solver iteration tuning.

---

## Quick audit checklist

```
[ ] Frame budget defined; profiling on target build
[ ] CPU vs GPU identified; hottest markers listed
[ ] No hot-path GC.Alloc / logs / GetComponent
[ ] Pooling for frequent spawn/despawn
[ ] Texture/mesh import overrides per platform
[ ] SRP Batcher / instancing / static batching reviewed
[ ] Lights/shadows/probes/LOD/occlusion profiled
[ ] Canvases split; raycasts trimmed
[ ] Physics matrix + colliders simplified
[ ] Audio load types correct; mono spatial
[ ] Animators not overused; Generic rig where possible
```
