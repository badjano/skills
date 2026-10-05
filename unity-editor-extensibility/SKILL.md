---
name: unity-editor-extensibility
description: >-
  Unity 6 editor extensibility: SerializedObject/SerializedProperty, UI Toolkit
  custom inspectors and property drawers, Scene View Overlays, and runtime
  World Space UI Toolkit. Use when writing Editor scripts, CustomEditor,
  PropertyDrawer, Overlay/ToolbarOverlay, UXML/USS, CreateInspectorGUI,
  CreatePropertyGUI, or migrating IMGUI editor tools to retained-mode UI.
---

# Unity Editor Extensibility (Unity 6)

Source paradigm: Unity Editor Deep Research — serialization bridge + retained-mode UI Toolkit. Complements `unity-architecture` (runtime) and `unity-canvas-ui-expert` (legacy uGUI Canvas).

**Default for new editor UI:** UI Toolkit (`CreateInspectorGUI` / `CreatePropertyGUI`). Do not start new inspectors in IMGUI `OnInspectorGUI` / `OnGUI`.

---

## 1. Serialization is the editor’s truth

The Editor does not inspect live C# object graphs for save/Inspector — it reads **serialized data** (YAML/binary) via the native↔managed bridge.

| Goal | API |
|------|-----|
| Safe edit with Undo / Prefab overrides / multi-select | `SerializedObject` + `SerializedProperty` |
| Dirty + flush to native | `ApplyModifiedProperties()` (or `WithoutUndo`) |
| Navigate tree | `FindProperty`, `FindPropertyRelative`, `Next`, `GetArrayElementAtIndex` |

### Hard rules

1. **Editor tools mutate via `SerializedObject`**, not reflection on live fields — Undo, multi-object edit, and Prefab override tracking depend on it.
2. **`ApplyModifiedProperties` bypasses C# property setters** — validation must live in `OnValidate`, custom drawers, or explicit apply-time checks, not only in setters.
3. **Multi-object edit:** writing a property writes **all** targets; reading returns the **first** target only. Handle `hasMultipleDifferentValues`.
4. Prefer property iteration over reflection — it hits the native serialized layout.

```csharp
var so = new SerializedObject(targets);
var prop = so.FindProperty("_speed");
EditorGUI.BeginChangeCheck(); // or UITK binding
prop.floatValue = newValue;
if (/* changed */) so.ApplyModifiedProperties();
```

---

## 2. UI Toolkit over IMGUI

| | IMGUI (legacy) | UI Toolkit (Unity 6 default) |
|--|----------------|------------------------------|
| Model | Immediate `OnGUI` every frame | Retained `VisualElement` tree |
| Structure / style | Imperative layout | UXML + USS (flexbox) |
| Data sync | Manual every repaint | Binding to `SerializedProperty` |
| Cost | High when complex | Redraw on change |

### Custom inspectors

- Derive `Editor`, override **`CreateInspectorGUI()`** → return root `VisualElement`.
- Bind with `PropertyField` / binding paths to `serializedObject` — do **not** poll every frame.

### Property drawers

- Override **`CreatePropertyGUI(SerializedProperty)`**, not `OnGUI`.
- `CreatePropertyGUI` runs **once** at tree build — sync via binding, not continuous code.

### Mixing ban (critical)

**A UI Toolkit `CreatePropertyGUI` drawer will not render inside an IMGUI `OnInspectorGUI` inspector** → “No GUI implemented”. Match stack:

- UITK inspector ↔ UITK drawers  
- IMGUI inspector ↔ IMGUI drawers  

When migrating, convert the **inspector first**, then drawers.

---

## 3. Scene View Overlays

Keep tools in the 3D view; avoid forcing designers into side panels.

1. Subclass `Overlay` or `ToolbarOverlay`.
2. Parameterless ctor + `[Overlay(...)]` attribute.
3. Override `CreatePanelContent()` → root `VisualElement`.

---

## 4. Runtime UI Toolkit (incl. World Space)

- Editor and runtime share VisualElement / UXML / USS.
- **World Space (Unity 6):** Panel Settings → Render Mode **`WorldSpace`** (`PanelRenderMode.WorldSpace`). Pre-6.2 APIs were experimental — do not invent Canvas-style workarounds when World Space exists.
- For **existing uGUI HUD**: keep using `unity-canvas-ui-expert`. Do not rewrite uGUI to UITK unless asked.

---

## 5. Asset Database / Addressables (editor-adjacent)

- Scenes/Prefabs are serialized hierarchies; nested prefabs/variants/overrides go through the Asset Database import pipeline.
- Prefer **Addressables** for runtime load/unload catalogs over raw AssetBundle plumbing in new work.
- Batch/import-heavy work belongs in Editor/`AssetPostprocessor` tools, not Play Mode hacks.

---

## Agent checklist

- [ ] New editor UI uses UI Toolkit, not IMGUI
- [ ] Edits go through `SerializedObject` / `SerializedProperty` + Apply
- [ ] Drawers use `CreatePropertyGUI`; no UITK drawer under IMGUI inspector
- [ ] Overlays registered with `[Overlay]` + `CreatePanelContent`
- [ ] World Space UITK uses Panel Settings WorldSpace (Unity 6+)
- [ ] Verified via `unity-mcp-skill` or `unity-cli` on the **target** project
---

## Related

- Runtime architecture / SerializeField: `unity-architecture`
- uGUI Canvas: `unity-canvas-ui-expert`
- Async in editor/play: `unity-awaitable`
- Tests: `unity-test-framework`
