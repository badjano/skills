# Unity Canvas UI Expert — Reference

Exhaustive rules from the project skill specification. Read when implementing SafeArea, layout matrices, ScrollViews, or MVP view templates.

## 2-Pass layout pipeline

1. **Pass 1 (bottom-up):** `ILayoutElement` computes `min` / `preferred` / `flexible` width and height.
2. **Pass 2 (top-down):** `HorizontalLayoutGroup`, `VerticalLayoutGroup`, `GridLayoutGroup`, `ContentSizeFitter` assign positions and sizes.

## Root CanvasScaler baseline

| Property | Value |
| --- | --- |
| UI Scale Mode | ScaleWithScreenSize |
| Reference Resolution | 1920×1080 (or 2560×1440) |
| Screen Match Mode | MatchWidthOrHeight |
| Match | Landscape ~0.5–1.0; Portrait ~0.0 |
| Reference Pixels Per Unit | 100 |

## Safe area pattern

```
[Screen Root Canvas]
 └── [Safe Area Container]  ← SafeAreaHandler
      ├── [Background Layer]  full-bleed, ignores safe area
      └── [Main UI Hierarchy] inside safe area
```

```csharp
using UnityEngine;

[RequireComponent(typeof(RectTransform))]
[DisallowMultipleComponent]
public sealed class SafeAreaHandler : MonoBehaviour
{
    private RectTransform _rectTransform;
    private Rect _lastSafeArea = Rect.zero;
    private Vector2Int _lastScreenSize = Vector2Int.zero;

    private void Awake()
    {
        _rectTransform = GetComponent<RectTransform>();
        ApplySafeArea();
    }

    private void Update()
    {
        if (_lastSafeArea != Screen.safeArea ||
            _lastScreenSize.x != Screen.width ||
            _lastScreenSize.y != Screen.height)
        {
            ApplySafeArea();
        }
    }

    private void ApplySafeArea()
    {
        _lastSafeArea = Screen.safeArea;
        _lastScreenSize = new Vector2Int(Screen.width, Screen.height);

        Vector2 anchorMin = Screen.safeArea.position;
        Vector2 anchorMax = Screen.safeArea.position + Screen.safeArea.size;

        anchorMin.x /= Screen.width;
        anchorMin.y /= Screen.height;
        anchorMax.x /= Screen.width;
        anchorMax.y /= Screen.height;

        _rectTransform.anchorMin = anchorMin;
        _rectTransform.anchorMax = anchorMax;
        _rectTransform.offsetMin = Vector2.zero;
        _rectTransform.offsetMax = Vector2.zero;
    }
}
```

## Anchor and layout matrix

- **Full stretch:** `anchorMin=(0,0)`, `anchorMax=(1,1)`; offsets `(left,bottom)` / `(-right,-top)`.
- **Point anchors:** `anchorMin == anchorMax`; size via `sizeDelta`.

| Goal | Container | Child | Notes |
| --- | --- | --- | --- |
| Expanding list | VLG + CSF vertical Preferred | LayoutElement | Child Force Expand Height = false |
| Proportional split | HLG Control+Expand | LayoutElement flexible 0.7 / 0.3 | minWidth = 0 |
| Aspect icon | parent layout | AspectRatioFitter + LayoutElement | FitInParent / EnvelopeParent |
| Expanding text card | VLG + CSF | TMP + LayoutElement | wrap or preferred sizes |

## ScrollView structure

```
[ScrollView_Root] ScrollRect + Image
 └── [Viewport] RectMask2D
      └── [Content] LayoutGroup + ContentSizeFitter
           └── items…
```

1. Prefer `RectMask2D` over `Mask` (stencil / extra draw calls).
2. Optional sub-canvas on Content for rebuild isolation.
3. Vertical Content: anchors `(0,1)–(1,1)`, pivot `(0.5,1)`; CSF vertical Preferred; VLG Control Child Size W/H true, Force Expand Height false.

## Prefab tiers

- Atoms: primary/secondary button, heading label, icon badge
- Molecules: labeled input, dropdown group, inventory slot row
- Organisms: confirmation modal, inventory screen

## MVP view template

```csharp
using TMPro;
using UnityEngine;
using UnityEngine.UI;

[DisallowMultipleComponent]
public sealed class ActionButtonView : MonoBehaviour
{
    [Header("Internal References")]
    [SerializeField] private Button button;
    [SerializeField] private TextMeshProUGUI labelText;
    [SerializeField] private Image iconImage;
    [SerializeField] private LayoutElement layoutElement;

    public Button.ButtonClickedEvent OnClick => button.onClick;

    public void SetData(string label, Sprite icon, bool enabled = true)
    {
        labelText.text = label;
        iconImage.sprite = icon;
        iconImage.gameObject.SetActive(icon != null);
        button.interactable = enabled;
    }

    public void OverrideWidth(float preferredWidth, float flexibleWidth = -1)
    {
        if (layoutElement == null) return;
        layoutElement.preferredWidth = preferredWidth;
        layoutElement.flexibleWidth = flexibleWidth;
    }
}
```

## Zero-allocation / lifecycle

- Do not read `rect.width/height` same frame after layout mutation without `ForceRebuildLayoutImmediate` once on the shared parent.
- Strip `raycastTarget` on decorative graphics (verify button hit graphics first).
- Ban TMP auto-sizing in dynamic lists.
- Do not use `image.material.color` (breaks batching).

## Prefab YAML (summary)

See [prefab-yaml.md](prefab-yaml.md) for fileID graph, stretch offset mapping, project panel map, and incident fixes.

When stretch anchors are active:

- `Left = offsetMin.x`, `Bottom = offsetMin.y`
- `Right = -offsetMax.x`, `Top = -offsetMax.y`

Always baseline with `git show develop:<prefab>` (or main) before recalculating anchors.
