# Mobile Optimization (Unity 2022 LTS)

Platform deltas for **iOS / Android**. Read [shared.md](shared.md) first, then apply this file.

Source: [Optimize your mobile game performance](https://unity.com/resources/optimize-mobile-game-performance-unity-2022lts).

---

## Mobile-first principles

1. **Thermal is a feature constraint.** Sustained max budget = throttle. Work at ~**65%** of frame budget.
2. Profile in **short bursts**; cool device **10–15 minutes** between sessions so heat doesn’t fake a regression.
3. Always test **min-spec and max-spec** devices you claim to support.
4. Prefer **30 fps** as default tradeoff (Unity mobile default); raise to 60 only where art/design needs it. Adjust `Application.targetFrameRate` per mode (menus lower, gameplay higher).
5. **VSync is effectively on** at hardware level even if Editor Quality disables it — missed GPU refresh holds the frame.

---

## Frame budget reminder

| Target | Working budget |
|--------|----------------|
| 30 fps | ~22 ms |
| 60 fps | ~11 ms |

Cutscenes/loading may exceed briefly; gameplay must not.

---

## Adaptive Performance (Samsung)

Package monitors thermal/power and can scale quality:

- Desired frame rate trend
- Temperature / proximity to thermal event
- CPU- vs GPU-bound

Use scalers (LOD bias, etc.) or automatic Indexer modes. **Samsung devices only** — still design thermal headroom for all OEMs.

---

## Project settings (mobile)

| Setting | Action |
|---------|--------|
| Accelerometer Frequency | Disable or lower if unused |
| Auto Graphics API | Off; strip unused APIs (fewer shader variants) |
| Quality levels | Keep only what you ship |
| Physics unused | Auto Simulation / Auto Sync Transforms off |
| targetFrameRate | Explicit; don’t assume Editor values |

---

## Assets (mobile)

- **ASTC** for modern iOS/Android min-spec.
  - Very old iOS (A7 and below): PVRTC
  - Pre-2016 Android: ETC2
- If ASTC unavailable and block compress quality fails: prefer **16-bit** over 32-bit RGBA.
- POT textures required for PVRTC/ETC-class formats.
- Aggressive Max Size reduction — non-destructive, huge memory win.
- Addressables + CDN (e.g. Cloud Content Delivery) for smaller install + DLC.

---

## Graphics (mobile)

### Pipeline

- Prefer **URP**; Forward or **Forward+** for more lights without classic Forward’s per-object limit.
- Deferred only when many dynamic lights justify G-buffer cost (watch normal encoding on mobile GPUs).
- Use URP Lit/Unlit; keep **shader variants/keywords** minimal for SRP Batcher + memory.

### Must-do GPU cuts

| Tip | Why |
|-----|-----|
| Avoid many dynamic lights (Forward) | Fill + CPU cost |
| Disable shadows when possible | Draw calls; use blob/fake shadows |
| Bake GI / lightmaps | Runtime free lighting |
| Light Probes for movers | Cheap vs realtime |
| LOD + Occlusion Culling | Less shaded geometry (profile cull CPU) |
| **Don’t run native resolution blindly** | `Screen.SetResolution` to balance quality/speed |
| Limit cameras | ~1 ms CPU each on low-end is common |
| Simple shaders; few variants | Memory + SetPass |
| Minimize alpha / overdraw | Fill-rate limited devices |
| Limit fullscreen post | Bloom/glow are expensive |
| Few Reflection Probes | Batch + bandwidth cost |
| Optimize SkinnedMeshRenderers | BakeMesh when static pose OK |

### Batching note

Draw-call reduction is **vital** on mobile — treat SRP Batcher + atlasing + shared materials as first-class.

### Mali / Arm

**System Metrics Mali** package: low-level GPU counters in Profiler / Recorder / CI on Arm GPUs.

---

## UI (mobile)

Shared UGUI/UI Toolkit rules apply harder:

- Multiple aspect ratios; Device Simulator / Xcode / Android Studio virtual devices.
- Alternate UI layouts per class of device when needed.
- Touch-heavy: fewer GraphicRaycasters, fewer Raycast Targets.

---

## Audio (mobile)

- SFX sample rate ≤ **22,050 Hz** usually enough.
- Load Type table in [shared.md](shared.md); streaming for music is mandatory for memory.

---

## Animation (mobile)

- Generic rig default unless Humanoid features required.
- Prefer legacy Animation / tweens over Animator spam on UI.
- Mecanim is costly — limit usage.

---

## Physics (mobile)

PhysX is expensive on handhelds:

- Match Fixed Timestep to target fps (e.g. `0.033` for 30 fps).
- Primitives over MeshColliders.
- Physics Debugger before adding complexity.

---

## Mobile audit extras

```
[ ] Working budget ~65% of hard ms budget
[ ] Profiled cool + warm device; min-spec covered
[ ] targetFrameRate intentional per game mode
[ ] Accelerometer disabled if unused
[ ] ASTC (or documented legacy format) on textures
[ ] Resolution scaled if GPU-bound at native
[ ] Cameras counted; shadows/dynamic lights justified
[ ] Adaptive Performance considered for Samsung SKUs
[ ] Overdraw checked (RenderDoc / Rendering Debugger)
```
