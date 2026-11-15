# Roblox UI Style Craft — Full Reference

Companion guide to `roblox-ui-style-craft`. Cross-reference `roblox-gui` for `ScreenInsets`, flex/grid layout fundamentals, safe zones, and gamepad focus rules.

---

## 1. Figma & Photoshop Asset Pipeline

### Why Transparent PNGs Get Dark Borders ("Alpha Bleed")
Figma and Photoshop write hidden RGB `(0, 0, 0)` into fully transparent (`alpha = 0`) pixels. When Roblox scales or rotates an `ImageLabel`, GPU bilinear filtering averages edge pixels with neighboring `alpha = 0` black pixels, leaving muddy dark outlines.

### Clean Export Checklist
1. **Use `Frame`, not `Rectangle`, in Figma:**
   - Nested clipping masks, vector SVGs, inner shadows/strokes, and blend modes (`Overlay`, `Soft Light`) behave predictably inside `Frame`s and stay editable for sliced exports.
   - Align coordinates and sizes to whole integers.
2. **Add 4px transparent outer margin:** Leave padding between vector art and the exported frame edge so UV clamping never stretches edge texels across transparent space.
3. **Export tintable chrome in `#FFFFFF`:** Export badges, sunbursts, soft glows, gloss strips, and asymmetric 9-slice shapes as pure white (`#FFFFFF`) so rarity colors and themes are driven in Studio via `ImageColor3` and `UIGradient`.
4. **Stay within `1024x1024`:** Roblox clamps uploaded textures above `1024px`, blurring fine vector details. Export icons at `2x` (`128x128` for `64px` HUD icons, `512x512` for portrait cards).
5. **Apply Edge Lighting & Shine in Figma:**
   - Duplicate the card/button frame without fill, add an inner stroke (`#FFFFFF`, soft falloff/opacity), and set Blend Mode to `Soft Light` or `Overlay` to give curved specular rim definition before flattening decorative overlays.
   - Confine diagonal shine bars and star accents inside a clipped transparent `Frame`.
6. **Always Alpha-Bleed before uploading:**
   - Photoshop's *Export As* and *Save for Web* discard color behind `alpha = 0`.
   - Run all exported transparent PNGs through an alpha-bleed utility (e.g., **`pixelfix`**) or save via SuperPNG with *Clean Transparent* unchecked over a dilated edge layer so `alpha = 0` pixels carry adjacent opaque RGB values.
7. **Configure 9-Slice accurately in Studio:**
   - For stretched panels or notched banners: set `ScaleType = Enum.ScaleType.Slice`, place `SliceCenter = Rect.new(left, top, right, bottom)` in **source image pixels** strictly inside the flat center band, and set `SliceScale = 0.5` for `2x` exports.

---

## 2. Cartoony UI & Shop Visual Hierarchy

Drawing flat rounded boxes looks amateur ("individuality + depth" comes from stacked micro-contrast and clear visual prioritization):

### 3D Extruded Cartoon Button Anatomy
Keep the interactive `TextButton` stationary so its hit area never shifts away from the cursor while pressing:
- **`BuyButton` (`TextButton`, stationary container):** `BackgroundTransparency = 1`, `Text = ""`, `AutoButtonColor = false`, `ClipsDescendants = false`.
- **`ExtrusionBase` (`Frame` or `UIShadow` with zero blur):** Bottom-anchored deep dark shade (e.g., `#115E34` under a bright `#22C55E` face), `UICorner` matching `Face`, `UIStroke` (`Border`, `Round`).
- **`Face` (`Frame`, size `UDim2.new(1, 0, 1, -ExtrusionPx)`):**
  - `UICorner` + saturated dark outer `UIStroke` (`Border`, `Round`).
  - Vertical `UIGradient` (`Rotation = 90`: lime top -> mid emerald -> deep green bottom).
  - **`InnerRim` (`Frame` inset 3–4px):** transparent fill + white `UIStroke` + vertical `UIGradient` fading white from `0.35` top transparency to `1.0` bottom transparency.
  - **`GlossPill` (`Frame` near top):** white rounded pill (`UICorner = 1, 0`), vertical `UIGradient` transparency `0.45 -> 0.92`.
  - **`PriceRow` (`Frame` + horizontal `UIListLayout`):** currency `ImageLabel` (`ScaleType.Fit`) + `TextLabel` with `UIStroke` (`ApplyStrokeMode.Contextual`, `Round`) and bold font.

### High-Converting Shop Modal Hierarchy
- **Backdrop contrast:** Dim behind modals (`ScreenInsets = None` full-bleed dark overlay) and pair contrasting top headers (saturated banner pill overlapping/capping the top edge) against a clean panel body (crisp light surface or deep translucent `#0D111A` obsidian).
- **Oversized Hero Card first:** Make the primary/featured bundle noticeably larger (`LayoutOrder = 1` or spanning multiple flex/grid units) with a distinct rarity gradient and corner shine so the eye lands immediately.
- **Unmistakable Buy CTA:** Green gradient CTA with inline currency icon outperforms plain text price labels on quick scan.

---

## 3. Roblox Stud UI Recipe

Classic Roblox stud styling blends procedural UI properties with a subtle repeating stud surface:
1. **Base Panel (`Frame`):** Rounded `UICorner`, saturated base color + subtle top-to-bottom `UIGradient`, and bold outer `UIStroke` (`StrokeSizingMode.ScaledSize`).
2. **Stud Texture Overlay (`ImageLabel` child, `Size = 1, 0, 1, 0`, `Active = false`, `ZIndex` above background):**
   - White stud silhouette asset (`128x128` seamlessly tileable PNG, alpha-bled).
   - `ScaleType = Enum.ScaleType.Tile`
   - `TileSize = UDim2.fromOffset(36, 36)` (or scaled proportionally per breakpoint).
   - `ImageTransparency = 0.82–0.90` with slight `UIGradient` fade so studs frame the edges softly without reducing text legibility.
   - Child `UICorner` matching parent radius (clipped cleanly or masked under `CanvasGroup` for static chrome).
3. **Top Bevel / Inner Lip:** Thin bright inset `UIStroke` (`BorderOffset` negative inset or inset rim `Frame`) + dark `ExtrusionBase` or crisp zero-blur `UIShadow`.

---

## 4. Modern 2026 Roblox UI Styling APIs

### `UIShadow` (June 2026 Full Release)
Replaces uploaded 9-slice shadow textures for rectangular/rounded frames:
- Properties: `Enabled`, `Color`, `Transparency`, `BlurRadius` (`UDim`, pixels or scale of shortest axis), `Spread` (`UDim2`), `Offset` (`UDim2`), `Inset` (`boolean`, pushes shadow inward for inner glow/bevel), and `ZIndex` (negative renders behind sibling content).
- Multiple `UIShadow` instances can be parented to one `GuiObject`, and a `UIGradient` child inside `UIShadow` creates multi-color gradient glows/shadows.
- **Caveats:** Not supported on `Path2D`; when parented to a `TextLabel`/`TextButton`/`TextBox`, `UIShadow` shadows the **rectangular box**, not the glyph outlines (use `UIStroke Contextual` on text glyphs).

### Per-Corner `UICorner` (June 2026 Full Release)
- Properties: `TopLeftRadius`, `TopRightRadius`, `BottomLeftRadius`, `BottomRightRadius` (`UDim`). Great for speech bubbles, attached category tabs, and top-only modal headers without external 9-slices.
- **Trap:** `CornerRadius` acts as a legacy alias—reading it returns `TopLeftRadius`, setting it overwrites all four corners, and Roblox Studio's Style Editor forbids mixing `CornerRadius` and per-corner tokens inside styling rules. Pick either `CornerRadius` OR the four explicit corner properties per rule.

### Upgraded `UIGradient` (Oct 2026 Full Release)
- Properties: `Type` (`Enum.GradientType`: `Linear`, `Radial`, `Conical`, `Elliptical`), `Scale`, `TileMode` (`Enum.GradientTileMode`: `Clamp`, `Repeat`, `Mirror`), and full `Path2D` support.
- `Radial` + `Elliptical` create spotlights/vignettes and glossy radial highlights without raster textures; `Conical` builds sunbursts and radial cooldown pies cleanly.
- **Caveat:** Check very dark gradients in Studio for subtle banding; adjust keypoint spacing or add tiny hue shifts across dark stops to smooth transitions.

### Proportional `UIStroke` (`StrokeSizingMode` & `BorderOffset`)
- Set `UIStroke.StrokeSizingMode = Enum.StrokeSizingMode.ScaledSize` so border thickness scales relative to the parent container's shortest dimension (or text size for `Contextual` strokes) rather than remaining hardcoded viewport pixels that bloat on phones.
- Use `BorderOffset` (`UDim`, pairs with `BorderStrokePosition`) for double-rim chrome, clean inner borders, or separated focus rings.

### `StyleSheet`, `StyleRule`, and `StyleQuery`
```lua
--!strict
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local sheet = Instance.new("StyleSheet")
sheet.Name = "ShopTheme"
sheet.Parent = ReplicatedStorage

-- Design tokens
sheet:SetAttribute("SurfacePrimary", Color3.fromRGB(244, 249, 255))
sheet:SetAttribute("StrokeDark", Color3.fromRGB(30, 58, 138))
sheet:SetAttribute("HeaderCorner", UDim.new(0, 20))
sheet:SetAttribute("FlatCorner", UDim.new(0, 0))

-- Tag rule + pseudo-instance targeting (use :: for modifiers, NOT CSS > UICorner)
local bodyRule = Instance.new("StyleRule")
bodyRule.Name = "ModalBody"
bodyRule.Selector = "Frame.ModalBody"
bodyRule:SetProperty("BackgroundColor3", "$SurfacePrimary")
bodyRule.Parent = sheet

local topHeaderCornerRule = Instance.new("StyleRule")
topHeaderCornerRule.Name = "HeaderCorners"
topHeaderCornerRule.Selector = "Frame.HeaderCap::UICorner"
topHeaderCornerRule:SetProperties({
	TopLeftRadius = "$HeaderCorner",
	TopRightRadius = "$HeaderCorner",
	BottomLeftRadius = "$FlatCorner",
	BottomRightRadius = "$FlatCorner",
})
topHeaderCornerRule.Parent = sheet

-- Proportional ScaledSize stroke
local cardStrokeRule = Instance.new("StyleRule")
cardStrokeRule.Name = "CardStroke"
cardStrokeRule.Selector = "Frame.ItemCard::UIStroke"
cardStrokeRule:SetProperties({
	ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
	LineJoinMode = Enum.LineJoinMode.Round,
	StrokeSizingMode = Enum.StrokeSizingMode.ScaledSize,
	Thickness = 0.035, -- 3.5% of shortest parent edge
	Color = "$StrokeDark",
})
cardStrokeRule.Parent = sheet

-- Built-in global queries work directly in Selector without creating a StyleQuery instance
local compactRule = Instance.new("StyleRule")
compactRule.Name = "SmallViewportRule"
compactRule.Selector = "Frame.ModalBody@ViewportDisplaySizeSmall"
compactRule.Priority = 10
compactRule:SetProperty("BackgroundColor3", Color3.fromRGB(250, 252, 255))
compactRule.Parent = sheet

-- Custom StyleQuery monitors its parent GuiBase2d (e.g., ScreenGui) and exposes @QueryName.
-- Use >> for descendants, and never mutate Size on the same GuiBase2d being queried (avoids feedback loops).
local wideQuery = Instance.new("StyleQuery")
wideQuery.Name = "WideGridQuery"
wideQuery:SetConditions({
	MinSize = Vector2.new(720, 0),
})
wideQuery.Parent = targetScreenGui -- Parent must be a GuiBase2d (not StyleSheet)

local wideRule = Instance.new("StyleRule")
wideRule.Name = "WideGridModal"
wideRule.Selector = "@WideGridQuery >> Frame.ModalBody"
wideRule.Priority = 10
wideRule:SetProperty("Size", UDim2.fromScale(0.78, 0.82))
wideRule.Parent = sheet
```
- When an element needs **multiple `UIStroke`s** (e.g., outer black comic outline + inner white rim), give each `UIStroke` a distinct name or tag (`.OuterStroke`, `.InnerRim`) and target them via separate `StyleRule` selectors.

---

## 5. Multi-Device Scaling & Typography Safety

1. **Drive responsive geometry inside `ScreenInsets.CoreUISafeInsets`:** Measure `ScreenGui.AbsoluteSize` (not `Camera.ViewportSize`) so topbar insets and phone notches are already removed. Keep full-screen dark dimmers in a separate `ScreenInsets.None` `ScreenGui` underneath.
2. **Prevent `ScrollingFrame` + `UIGridLayout` Y-Scale overlap bugs:**
   - Never use `Y.Scale > 0` in `CellSize` or `CellPadding` when `ScrollingFrame.AutomaticCanvasSize = Enum.AutomaticSize.Y`.
   - Set `VerticalScrollBarInset = Enum.ScrollBarInset.Always` and recompute exact pixel `CellSize` (`UDim2.fromOffset(w, math.floor(w * aspect))`) whenever `AbsoluteWindowSize` changes, OR use Studio conversion tooling (**UI Plus** plugin: `With Children` -> `Toggle Aspect Ratio Constraint` -> `Offset to Scale`) for static aspect-locked HUD widgets.
3. **Flex Layouts (`UIListLayout.Wraps` + `UIFlexItem`):** Combine `HorizontalFlex` / `VerticalFlex` (`Fill`, `SpaceBetween`, `SpaceEvenly`) with `UIFlexItem` (`GrowFactor`, `ShrinkFactor`) and `UISizeConstraint` (`MinSize`, `MaxSize`) to build split inspector + grid shops cleanly without manual anchor math.
4. **Device Emulator Audit:** Always test in **Test -> Device Emulator** across Desktop `1920x1080`, Tablet, and Phone Landscape (`~375–390px` total height, `~310–335px` safe height). At Phone Landscape height, collapse multi-column side inspectors into full-width grids or tap drawers.
5. **Fonts & Text Guardrails:**
   - `FredokaOne` only ships `Enum.FontWeight.Regular`; use `BuilderSans` or `Nunito` when you need `SemiBold`/`Bold`/`ExtraBold`.
   - Avoid `TextScaled = true` on repeating card names (causes mismatched sizes per card); use consistent `TextSize` + `TextTruncate = Enum.TextTruncate.AtEnd`. Only use `TextScaled = true` inside flexible localized CTA buttons paired with `UITextSizeConstraint`.
