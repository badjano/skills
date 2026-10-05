---
name: unity-render-graph
description: >-
  Unity 6 URP/HDRP Render Graph API: declarative raster/compute passes,
  transient resource aliasing, pass culling, and RecordRenderGraph patterns.
  Use when writing ScriptableRenderPass, ScriptableRendererFeature, custom
  post-processing, blit passes, compute in the pipeline, migrating from
  imperative CommandBuffer/RTHandle passes, or debugging Render Graph errors.
---

# Unity 6 Render Graph

Source paradigm: Unity Editor Deep Research — declarative GPU pass graph with topological memory aliasing. Complements `unity-optimization` (profile-first) and `unity-architecture`.

**Default for new URP/HDRP custom passes in Unity 6:** implement via **Render Graph** (`RecordRenderGraph`), not legacy imperative `Execute` + manual `RTHandle` allocation.

---

## Why it exists

Legacy `ScriptableRenderPass` + manual `CommandBuffer` / RTHandles caused VRAM fragmentation, redundant barriers, and fragile mobile/desktop variance.

Render Graph: **declare** inputs/outputs during setup; the compiler:

1. **Maps producer → consumer** dependencies (can reorder)
2. **Culls** passes whose outputs are unused (and not backbuffer/persistent)
3. **Aliases** non-overlapping transient resources onto the same VRAM

Barriers are inserted from the read/write map — do not hand-roll them for raster builder passes.

---

## Implementation flow

Inside `ScriptableRenderPass.RecordRenderGraph(RenderGraph renderGraph, ContextContainer frameData)`:

1. `renderGraph.AddRasterRenderPass<PassData>("Pass Name", out var passData)`
2. Fill `PassData` with **handles / settings only** (no physical RT access yet)
3. Declare attachments: `builder.SetRenderAttachment(...)` (and reads as required)
4. `builder.SetRenderFunc((PassData data, RasterGraphContext ctx) => { ... })`

```csharp
public override void RecordRenderGraph(RenderGraph renderGraph, ContextContainer frameData)
{
    using var builder = renderGraph.AddRasterRenderPass<PassData>("MyPass", out var passData);
    // passData.input = ...; passData.material = ...;
    builder.SetRenderAttachment(/* color handle */, 0);
    builder.SetRenderFunc(static (PassData data, RasterGraphContext ctx) =>
    {
        // Resolve handles → real GPU resources ONLY here
        // Blit / DrawProcedural / etc. via ctx.cmd
    });
}
```

### Hard rules

| Rule | Why |
|------|-----|
| Declare **all** attachments in setup — no swapping targets in the func | Graph needs fixed output schema |
| Access physical resources **only** inside `SetRenderFunc` | Transients are not allocated during record |
| Prefer transient graph resources over manual RTHandle pools | Enables aliasing + culling |
| Use **UnsafePass** / compatibility only for unmigrated plugins | Sacrifices automatic memory guarantees |
| Compute: use graph compute passes sharing handles with raster | Avoids manual barriers between stages |

---

## 2D / specialized notes

- 2D URP integrates via passes such as `DrawRenderer2DPass` on the same graph — custom 2D lighting/post should follow graph attachment rules, not old blit-only paths.
- XR: prefer pipeline foveated-rendering support over custom full-res hacks when available.
- Ray tracing / DLSS: use engine `RayTracingAccelerationStructure` / `DLSSContext` APIs inside graph-compatible execution, not ad-hoc backbuffer grabs.

---

## Migration from legacy passes

1. Move allocation declarations into `RecordRenderGraph` builder setup.
2. Move draw/blit into `SetRenderFunc`.
3. Delete manual “create RT every frame” / overlapping RTHandle lifetimes.
4. Profile before/after (`unity-optimization`) — expect lower transient VRAM, not always lower CPU if setup is chatty.

---

## Agent checklist

- [ ] Custom pass uses `RecordRenderGraph` + raster/compute builder
- [ ] No RTHandle / resource use outside `SetRenderFunc`
- [ ] Attachments declared up front; no dynamic SetRenderTarget in hot path
- [ ] Unsafe/compatibility mode only with explicit justification
- [ ] Verified compile on target project (`unity-mcp-skill` / `unity-cli`)

## Related

- Frame budgets / GPU bound diagnosis: `unity-optimization`
- Editor tooling: `unity-editor-extensibility`
