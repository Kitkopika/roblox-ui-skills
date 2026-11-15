---
name: roblox-ui-style-craft
description: Use when styling Roblox UI (cartoon bevels, stud textures, glass, or modern themes), exporting Figma or Photoshop UI assets without dark edge haloes, using StyleSheet, StyleRule, StyleQuery, UIShadow, per-corner UICorner, UIGradient types, or UIStroke ScaledSize, or fixing squished cards and giant strokes across desktop, tablet, and phone.
---

# Roblox UI Style Craft

Build intentional Roblox UI with layered depth, clean asset exports, native 2026 styling APIs, and cross-device scaling. Read `roblox-gui` first for `ScreenInsets`, `Scale` vs `Offset`, and gamepad focus; read [references/full.md](references/full.md) for Figma/Photoshop checklists, cartoon/stud hierarchies, and `StyleSheet` code.

## Non-Negotiable Rules

1. **Alpha-bleed transparent PNGs before Studio upload.** Figma and Photoshop write `(0,0,0)` RGB into `alpha = 0` pixels, creating dark outlines under bilinear scaling. Run PNGs through `pixelfix`/alpha-bleed and keep textures `<= 1024x1024` (Studio's clamp).
2. **Design in Figma `Frame`s, not `Rectangle`s** so clipping, SVG icons, blend-mode shine (`Soft Light`/`Overlay`), and **Edge Lighting** (duplicate invisible `Frame` + inner stroke + blend mode) compose cleanly. Export tintable chrome (badges, gloss, shadows, 9-slices) in `#FFFFFF` (`4px` outer padding) and color via `ImageColor3` + `UIGradient`.
3. **Use native 2026 primitives instead of legacy hacks:**
   - **`UIShadow`:** Use `BlurRadius` (`UDim`), `Spread` (`UDim2`), `Offset`, `Inset`, and negative `ZIndex` (even with a child `UIGradient`) instead of 9-slice shadow PNGs. Caveats: no `Path2D`; text shadows box bounds, not glyphs (use `UIStroke Contextual` for text).
   - **Per-corner `UICorner`:** Set `TopLeftRadius`, `TopRightRadius`, `BottomLeftRadius`, `BottomRightRadius`. Never mix `CornerRadius` with per-corner tokens in a `StyleSheet` rule.
   - **Proportional `UIStroke`:** Set `StrokeSizingMode = Enum.StrokeSizingMode.ScaledSize` (+ `BorderOffset`) so phones avoid oversized borders.
   - **`UIGradient`:** Use `Type` (`Linear`, `Radial`, `Conical`, `Elliptical`), `Scale`, and `TileMode` (`Enum.GradientTileMode`: `Clamp`, `Repeat`, `Mirror`); check dark ramps for banding.
4. **Use Roblox `StyleSheet` selectors (`::`, `>`, `>>`, `@`).** Target modifiers via `::` (`.Card::UICorner`), use `>` for direct child and `>>` for descendant matching (not CSS spaces), name/tag multiple `UIStroke`s distinctly, parent `StyleQuery` (`MinSize`, `MaxSize`, `AspectRatioRange`) under its monitored `GuiBase2d` (or `::StyleQuery` in `StyleSheet`); built-in `@ViewportDisplaySizeSmall`, `@PreferredInput*`, `@PreferredTextSize*`, and `@ReducedMotionEnabledTrue` need no `StyleQuery`.
5. **Prevent scroll overlap and card squish.** Never mix `UIGridLayout` `Y.Scale` with `ScrollingFrame.AutomaticCanvasSize = Y`—compute pixel `CellSize` from `AbsoluteWindowSize.X` with `VerticalScrollBarInset = Always`, or pair `UIAspectRatioConstraint` + `Scale` sizing (batch-convertible in Studio with UI Plus `With Children`), verifying in **Device Emulator**.
6. **Guard typography.** `FredokaOne` only supports `Regular` (use `BuilderSans` or `Nunito` for mixed weights). Standardize repeating labels with `TextSize` + `TextTruncate.AtEnd`, and pair any `TextScaled = true` button with `UITextSizeConstraint`.

## Quick Reference

| Goal / Symptom | Fix | Reference |
| :--- | :--- | :--- |
| Dark halo on transparent PNGs | Alpha-bleed (`pixelfix`), 4px margin, `#FFFFFF` exports | [references/full.md](references/full.md) §1 |
| Flat dull cartoon UI / shop cards | Extrusion + gradient face + gloss + edge lighting + contrast | [references/full.md](references/full.md) §2 |
| Tactile Roblox Stud UI | `ScaleType.Tile` + `TileSize` overlay + `UIGradient` + `UIStroke` | [references/full.md](references/full.md) §3 |
| Broken `StyleSheet` / giant mobile strokes | `::` selectors + `StrokeSizingMode.ScaledSize` + `StyleQuery` | [references/full.md](references/full.md) §4–5 |
