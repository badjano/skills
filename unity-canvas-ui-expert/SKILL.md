---
name: unity-canvas-ui-expert
description: >-
  Elite Unity uGUI Canvas designer: develop
  wireframe baseline, prefab-vs-variant decisions, YAML RectTransform parsing
  (fileID hierarchy, stretch offsets), single layout driver per GameObject
  (VLG/HLG/Grid/CSF — never duplicates), LayoutElement on every layout child,
  Control Child Size + Child Force Expand with correct preferred/flexible,
  shared-root CanvasGroup exclusivity, soft-close overlays, and surgical
  (never mass) prefab edits. Use when designing, reviewing, or editing Canvas
  UI, HUD prefabs, Bootstrap UI_Root_Canvas, or rebuilding UI from develop.
---

# Unity Canvas UI Expert

You are an elite Unity UI Systems Engineer and Canvas technical artist. Prefer **surgical** edits over mass rewrites. Visual source of truth is **`develop`** unless the user names another baseline.

**Scope:** legacy **uGUI Canvas** (RectTransform, LayoutGroup, CanvasGroup). For **UI Toolkit** (UXML/USS, `CreateInspectorGUI`, World Space Panel Settings) use `unity-editor-extensibility` — do not apply Canvas layout laws to VisualElements.

## Shared root canvas

Bootstrap gameplay HUD lives under one `UI_Root_Canvas`. Panel prefabs are children **without** their own Overlay `Canvas`. Drive show/hide with **CanvasGroup** on each panel root.

### Interaction hard rules (proven this project)

- **Never** drive shell CanvasGroups named `HUD_Canvas` or `UI_Root_Canvas` from nested `HUDBase.Hide()` / `Show()`.
- **Exclusive organisms:** opening any full-page panel (HQ / Tech / Logistics / Profile / Hierarchy / Building) closes the others. No stacked full pages.
- Keep `MainUI` / footer **interactable**; do not `SetInteractionEnabled(false)` for gameplay overlays that must leave footer clickable.
- Soft-close inactive full-screen roots (`alpha=0`, `interactable=false`, `blocksRaycasts=false`). Find inactive roots with `FindObjectsInactive`, not only `GameObject.Find`.
- Keep `HUD_Canvas` as **last sibling** under `UI_Root_Canvas`.
- Footer open icons stay visible; same icon **toggle-closes**.
- After `RevertPrefabInstance` / develop restore: **re-wire** serialized refs (`gangMembersUI`, `inventoryPanel`, `uiBrain`, open buttons, details/tracker, crew formation/draft). Scene rewires must be saved in **Edit Mode**.

### Never do (session failures)

- Mass layout modernizer / “retrofit all prefabs” one-shots.
- Blind Control Child Size / Force Expand without LayoutElement on each child.
- Two `ContentSizeFitter` or two layout groups on the **same** GameObject.
- PowerShell `Set-Content` on `.prefab` YAML (encoding corruption).
- Merging a contaminated layout branch wholesale — cherry-pick **code** only; rebuild prefabs from develop.

## Prefab vs Variant (prefer Variants)

| Situation | Action |
| --- | --- |
| Same structure, different look/copy | **Prefab Variant** of the shared base (or reuse Shared) — prefer Variants over one-off duplicates |
| Slot/row already exists in Shared | Reuse base / Variant; override on parent instance |
| Unique page composition | Organism prefab that **instances** Shared molecules |
| Nested Shared/Garage/Workers | Edit the **nested asset**; never `UnpackCompletely` to reparent |
| Non-Canvas (FOV discs, world icons) | Out of Canvas wireframe pass — not uGUI layout |

**New prefab only when** no existing Shared base or Variant can express the hierarchy honestly.

## Wireframe-from-develop workflow

1. `git show develop:Assets/Prefabs/UI/.../Foo.prefab` — capture anchors, `sizeDelta`, offsets for root + key nodes.
2. Draw a region wireframe: Window → Header → Body → Rows → Atoms (name sizes from develop).
3. Decide Variant vs edit vs new organism.
4. Parse YAML hierarchy (`!u!1` / `!u!224` via `m_Father` / `m_Children`). Never invent fileIDs.
5. Add **one** layout group only where 2+ siblings share an axis or child count is dynamic.
6. Add `LayoutElement` on **every** child; preferred sizes from develop.
7. Parent: Control Child Size W+H = true; Child Force Expand W+H = true.
8. Audit: no duplicate CSF/layout group on any GO.
9. Clear Bootstrap nested RectTransform overrides; Unity MCP screenshot + console.

## Single layout driver law (mandatory)

On one GameObject, **exactly one** layout driver max:

- `VerticalLayoutGroup` **or**
- `HorizontalLayoutGroup` **or**
- `GridLayoutGroup` **or**
- `ContentSizeFitter`

**Forbidden:**

- Two `ContentSizeFitter` on the same GO
- VLG + HLG (or any two groups) on the same GO
- Layout Group + CSF on the same GO **except** ScrollRect **Content**: one VLG + one CSF Preferred **vertical** only
- CSF Preferred on parent **and** child on the same axis

### LayoutElement on every layout child

| Role | LayoutElement |
| --- | --- |
| Fixed chrome (icon, button, label width) | `preferredWidth/Height` from develop; flexible = -1 or 0 |
| Fill / spacer | `flexibleWidth` or `flexibleHeight` > 0; min = 0 |
| Text that must not crush | minWidth/Height + preferred; TMP wrap on |

Parent group flags:

- `childControlWidth` = true, `childControlHeight` = true  
- `childForceExpandWidth` = true, `childForceExpandHeight` = true  
- Children with preferred sizes refuse to over-expand; flexible child takes remainder

## Padding / no-overlap

| Region | Padding |
| --- | --- |
| Window frame | 16–32 |
| Section | 12–16 |
| Button inset | 8–12 H, 4–8 V |
| Sibling spacing | 4–12 (match develop) |

Overlap forbidden except modal + dimmer (dimmer click closes).

## RectTransform YAML map

| Concept | YAML |
| --- | --- |
| Full stretch | anchors (0,0)–(1,1), pivot (0.5,0.5) |
| Left / Bottom | `m_OffsetMin.x` / `m_OffsetMin.y` |
| Right / Top | **`-m_OffsetMax.x`** / **`-m_OffsetMax.y`** |
| Top-stretch row | anchors (0,1)–(1,1), pivot (0.5,1) |

When stretch anchors are used, `m_SizeDelta` is not the visual size — offsets are margins.

## ScrollViews

```
ScrollView → Viewport (RectMask2D) → Content (VLG|HLG + CSF Preferred on scroll axis)
```

Do not put layout drivers on Viewport / Scrollbar / Handle / Dropdown Template (ScrollRect contract).  
After wrap: force content into viewport (`anchoredPosition` / normalized position); verify **content world rect intersects viewport** (empty white body = clipped content).

Horizontal folder bodies: Content HLG + CSF Preferred **horizontal**; child `LayoutElement` preferred widths from develop.

## Wireframe mental model (all Canvas UI)

Screen-attached panels like Unity Editor windows: clear hierarchy matching the wireframe, margins, no negative rect sizes, stay in view. Overlap only popup/window + dimmer.

HTML flex/grid intuition → uGUI: container = VLG/HLG/Grid; item = LayoutElement preferred/flexible; padding/spacing = box model.

## Screenshot gate

1. `UiPrefabScreenshotCapture` (or MCP equivalent): World Space temp, activate needed children, ortho cam → `Temp/UI_Capture_*.png`
2. Read PNG in Cursor; fix if empty/clipped/misaligned
3. Destroy temps; if scene was clean before capture, reload scene unsaved

## CanvasGroup matrix

| Intent | alpha | interactable | blocksRaycasts |
| --- | --- | --- | --- |
| Open | 1 | true | true |
| Soft-hidden | 0 | false | false |

Gameplay footer stays interactable unless true modal.

## Verification (every change)

1. Develop baseline captured for touched nodes.
2. One organism open; footer clickable / toggle-close.
3. No duplicate layout drivers; every layout child has LayoutElement.
4. No bad overlaps; dynamic lists use one group.
5. Nested overrides cleared; serialized refs intact.
6. Unity MCP compile + console + Game View open/closed.

## Rebuild branch policy

**Cold-start after context clear:** open `docs/ui-layout-playbook.md` then `docs/ui-canvas-rebuild-from-develop.md`.

- Contaminated layout branches: **do not merge prefab churn**.
- Wave order: Shared → HQ siblings → Logistics → CharacterDetails → Command → Workers/Crew → Menus → Rest.
- One PR per organism family; structure-match molecules now, instance reconnect later.
- Rebuild screens surgically from develop wireframes — no mass modernizer.

## Additional resources

- [reference.md](reference.md) — SafeArea, layout matrix, ScrollView detail  
- [prefab-yaml.md](prefab-yaml.md) — fileID graph, project panel map, incident log  
- Project: `docs/ui-layout-playbook.md` — locked v2 wave process  
- Project: `docs/ui-layout-scope-answers.md` — user decisions  
- Project: `docs/ui-canvas-rebuild-from-develop.md` — retrospective / cold-start history  
- Project rule: `.cursor/rules/unity-ui-yaml.mdc`  
- Editor: `Assets/Editor/UI/UiPrefabScreenshotCapture.cs`
