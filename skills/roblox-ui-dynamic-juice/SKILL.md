---
name: roblox-ui-dynamic-juice
description: Use when animating Roblox UI, implementing hover, press, and release button micro-interactions inside UIListLayout or UIGridLayout, using spr velocity springs, StyleRule SetPropertyTransition, animated UIGradient sweeps, CanvasGroup fades or blur fixes, layered reveal choreography, UI SFX polish, or ReducedMotionEnabled accessibility.
---

# Roblox UI Dynamic Juice

Build responsive, interruptible 2D UI motion without layout jitter, `CanvasGroup` re-rasterization stalls, or double-clicks during fades. Read `roblox-gui` for input basics and `roblox-ui-style-craft` for static tokens; read [references/full.md](references/full.md) for `spr`, shimmer, SFX, and `CanvasGroup` code recipes.

## Non-Negotiable Rules

1. **Never animate layout-owned `Size` or `Position`.** Changing `Size` inside `UIListLayout` forces sibling jitter and `UIGridLayout` overwrites `CellSize`. Use **`Hitbox -> Visual + UIScale`**: keep the layout-sized `GuiButton` (`Hitbox`, `AutoButtonColor = false`, `ClipsDescendants = false`) stationary so hitbounds never flicker, and scale an inner centered `Visual` (`AnchorPoint = (0.5, 0.5)`) via **`UIScale`**. Lift the slot `ZIndex` under `ZIndexBehavior.Sibling` on hover/select.
2. **Pick the right motion engine over chained tween races:**
   - **Simple state polish:** Use `StyleRule:SetPropertyTransition(prop, tweenInfo)` on `:Hover` / `:Press` targeting `::UIScale` or color. Caveat: `RepeatCount` and `Reverses` are ignored.
   - **Interruptible tactile juice:** Use **`spr.target(obj, dampingRatio, frequency, props)`** (and `spr.stop`) so fast enter -> leave -> click preserves velocity cleanly and blends `Color3` in CIELUV without `Completed:Once` token races.
   - **Choreographed sweeps:** Use `TweenService` (never deprecated `:TweenPosition`/`:TweenSize`) or a shared `RunService.PreRender` loop for `UIGradient.Offset`.
3. **Never animate 60Hz children inside an active `CanvasGroup`.** `CanvasGroup` needs `ZIndexBehavior.Sibling`, always forces `ClipsDescendants = true` (inset panels so borders/`UIShadow` do not clip), downscales on low graphics settings (blurring text), degrades above `2048px`, and **re-rasterizes its entire offscreen texture every frame if any descendant mutates**. Keep `CanvasGroup` solely for brief open/close `GroupTransparency` transitions (`Interactable = false` immediately on close; `Visible = false` when closed) and keep continuous shimmers/timers outside resting `CanvasGroup`s.
4. **Layer motion for physical weight.** Translate outer containers cleanly while clipped inner frames animate local sparkles/gleams, staggering shadows `~0.05–0.10s` behind main cards.
5. **Respect `ReducedMotionEnabled` and touch behavior.** Support `@ReducedMotionEnabledTrue` in `StyleSheet` and `GuiService.ReducedMotionEnabled` in Luau (collapse bounces/zooms/shimmers to quick fades); ignore hover scaling on `UserInputType.Touch`, keep `SelectionGained`/`SelectionLost` gamepad feedback, and reset stuck presses via `UserInputService.InputEnded`.
6. **Protect UI SFX from spam and clipping.** Preload sounds (`ContentProvider:PreloadAsync`), debounce hover triggers (`~0.035s`), vary `PlaybackSpeed` (`±3–5%`), and clone/clean transient sounds so rapid clicks never cut off audio tails.

## Motion Selector

| Effect | Mechanism | Trap Avoided |
| :--- | :--- | :--- |
| Button hover/press states | `spr.target` on `Visual.UIScale` or `SetPropertyTransition` | Hitbox edge-jitter; layout reflow; mid-tween snap |
| Badge diagonal gleam | Shared `PreRender` `UIGradient.Offset` inside unrotated `UICorner` frame | Rectangular `ClipsDescendants` clipping corners; tween churn |
| Modal open/close fade + scale | `CanvasGroup` + `UIScale`, `Interactable = false` on close start | Idle GPU VRAM retention; clicks during fade-out |
| Active shimmers / live timers | Host outside resting `CanvasGroup` (or unwrap after open fade) | Per-frame full-modal offscreen re-rasterization |
