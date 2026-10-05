# Unity Prefab YAML + project panel map

## Multi-document YAML graph

Unity prefabs are multi-document YAML. Objects link via `fileID`:

1. Find `GameObject` (`!u!1`) by `m_Name`.
2. Follow `m_Component` → `RectTransform` (`!u!224`), `CanvasGroup` (`!u!225`), `MonoBehaviour` (`!u!114`), etc.
3. Build tree: `m_Children` / `m_Father` on RectTransforms.
4. Nested instances use `stripped` + `PrefabInstance` modifications — Bootstrap overrides can hide prefab fixes until cleared.

### Safe edit rules

- Never renumber or invent fileIDs; never break `--- !u!<class> &<id>`.
- Prefer Unity MCP / PrefabUtility for adding/removing components.
- YAML only for surgical number/anchor edits when fileIDs stay valid.
- Restore: `git checkout develop -- path/to.prefab` (never PowerShell `Set-Content`).

## Baseline query

```bash
git show develop:Assets/Prefabs/UI/.../Foo.prefab
```

Capture: `m_AnchorMin/Max`, `m_Pivot`, `m_SizeDelta`, `m_AnchoredPosition`, `m_OffsetMin/Max`.

## Duplicate-driver audit

Before saving a prefab, ensure no GameObject has:

- Two `ContentSizeFitter`
- Two of `VerticalLayoutGroup` / `HorizontalLayoutGroup` / `GridLayoutGroup`
- Layout group without `LayoutElement` on each child

Exception: Scroll **Content** may have one VLG + one CSF Preferred vertical.

## `UI_Root_Canvas` organisms

| Panel root | Soft-close when inactive |
| --- | --- |
| `HUD_Canvas` | **Never** zero from nested HUDBases |
| `HUD_Headquarters` | yes (`HQ_UI` inactive) |
| `Building_Canvas` | yes (BuildingUINew inactive) |
| `HUD_Logistics` / `HUD_ProfileView` / Hierarchy | yes when closed |

Footer: keep MainUI interactable; exclusive open; `HUD_Canvas` last sibling; icons toggle-close.

## Prefab vs Variant

- Similar structure → Variant / Shared reuse  
- New prefab only when hierarchy cannot be expressed by existing Shared molecules  

## Layout group policy

| Situation | Use |
| --- | --- |
| 2+ siblings one axis / dynamic count | One VLG or HLG or Grid |
| Each child | LayoutElement preferred from develop |
| Parent flags | Control Child Size W+H + Child Force Expand W+H |
| Scroll Content | VLG + CSF Preferred vertical |
| Absolute overlays | no layout inject |

## Incident log

| Symptom | Cause | Fix |
| --- | --- | --- |
| Footer disappears on HQ | Nested Hide zeroed `HUD_Canvas` | Shell guard in `HUDBase` |
| Footer unclickable | MainUI blocksRaycasts false + overlay CG | Keep MainUI on; soft-close overlays |
| HQ icon gone | `ShowOpenButton(false)` | Keep visible / toggle-close |
| Squeezed HQ | Mass retrofit | Restore develop; surgical layout only |
| Prefab look unchanged | Bootstrap RectTransform overrides | Clear nested overrides |
| Manager + does nothing | `gangMembersUI` null | Re-wire to HQ WorkersListGroupImage |
| HQ open NREs | Missing inventoryPanel / crewDraft | Re-wire; null-guard Awake/OnEnable |
| Stacked pages | No exclusive close | `UIBrain.CloseExclusiveGameplayOrganisms` |
| Contaminated branch | Mass prefab YAML | New branch from develop; cherry-pick code only |
