# Roblox UI Dynamic Juice — Full Reference

Companion guide to `roblox-ui-dynamic-juice`. Cross-reference `roblox-ui-style-craft` for `StyleSheet` visual recipes and `roblox-gui` for gamepad focus and container sizing.

---

## 1. Layout-Safe Button Architecture (`Slot -> Hitbox -> Visual`)

Scaling a button inside `UIListLayout` or `UIGridLayout` directly creates two bugs: neighbors jump as layout recalculates, and shrinking on press moves the button edge out from under the cursor (`MouseEnter`/`MouseLeave` oscillation).

### Required Hierarchy
```text
Container (Frame with UIListLayout / UIGridLayout + UIPadding >= scale overshoot)
└── Slot (Frame | BackgroundTransparency=1 | ClipsDescendants=false | ZIndex=base)
    └── Hitbox (TextButton | Size=1,0,1,0 | BackgroundTransparency=1 | Text="" | AutoButtonColor=false | ClipsDescendants=false)
        └── Visual (Frame | AnchorPoint=0.5,0.5 | Position=0.5,0,0.5,0 | Size=1,0,1,0 | Active=false | Interactable=false)
            ├── UIScale (Scale = 1)
            ├── UICorner
            └── UIStroke
```
- **Why `Visual` stays inside `Hitbox` (or sibling under `Slot`):** Because `Hitbox` stays `1,0,1,0` at rest inside `Slot`, scaling `Visual.UIScale` never alters `Hitbox` bounds (avoid putting `UIScale` on `Hitbox` itself!), AND placing `Visual` inside `Hitbox` lets `StyleRule` target `"TextButton.JuiceBtn:Hover > Frame.Visual::UIScale"` directly without Lua when desired.
- Always add container `UIPadding` and layout cell gaps equal to or larger than maximum scale overshoot (`(maxScale - 1) * maxDim * 0.5`) so popped cards never clip against `ScrollingFrame` edges.

---

## 2. Choosing Between `StyleRule:SetPropertyTransition`, `spr`, and `TweenService`

### Option A: Native Styling Transitions (July 2026) — Best for Uniform State Polish
```lua
--!strict
-- NOTE: StyleRule transitions ignore TweenInfo.RepeatCount and TweenInfo.Reverses.
local hoverScaleRule = Instance.new("StyleRule")
hoverScaleRule.Name = "JuiceButtonHoverScale"
hoverScaleRule.Selector = "TextButton.JuiceButton:Hover > Frame.Visual::UIScale"
hoverScaleRule:SetProperty("Scale", 1.07)
hoverScaleRule:SetPropertyTransition("Scale", TweenInfo.new(0.18, Enum.EasingStyle.Back, Enum.EasingDirection.Out))
hoverScaleRule.Parent = sheet

local pressScaleRule = Instance.new("StyleRule")
pressScaleRule.Name = "JuiceButtonPressScale"
pressScaleRule.Selector = "TextButton.JuiceButton:Press > Frame.Visual::UIScale"
pressScaleRule:SetProperty("Scale", 0.93)
pressScaleRule:SetPropertyTransition("Scale", TweenInfo.new(0.07, Enum.EasingStyle.Quad, Enum.EasingDirection.Out))
pressScaleRule.Parent = sheet

-- Automatically disable spring/back overshoot when player enables Reduced Motion
local reducedRule = Instance.new("StyleRule")
reducedRule.Name = "JuiceButtonReducedMotion"
reducedRule.Selector = "TextButton.JuiceButton@ReducedMotionEnabledTrue:Hover > Frame.Visual::UIScale"
reducedRule.Priority = 20
reducedRule:SetProperty("Scale", 1)
reducedRule:SetPropertyTransition("Scale", TweenInfo.new(0))
reducedRule.Parent = sheet
```

### Option B: Velocity-Preserving Spring Motion (`spr`) — Best for Juicy Tactile Feel
Unlike chained `TweenService` calls (where cancelling a press tween and spawning a release tween loses velocity and requires fragile `animToken` guards), Fraktality's **`spr`** (`spr.target(instance, dampingRatio, frequency, goals)` + `spr.stop`) integrates continuous spring velocity across interruptions and blends `Color3` in perceptually uniform CIELUV. Never drive the same property (such as `UIScale.Scale`) with both `StyleRule:SetPropertyTransition` and `spr` simultaneously:
- `dampingRatio < 1` (`0.45–0.68`, `frequency = 18–26`): bouncy overshoot ideal for hover pops and reward bursts.
- `dampingRatio == 1` (`1.0`, `frequency = 16–22`): crisp critical settle for sliding drawer tabs and counters.
- `dampingRatio > 1`: gentle eased settling.

```lua
--!strict
local GuiService = game:GetService("GuiService")
local UserInputService = game:GetService("UserInputService")
-- Require project's spr module (github.com/fraktality/spr)
local spr = require(script.Parent.spr)

export type JuicyButtonOptions = {
	HoverScale: number?,
	PressScale: number?,
	OnActivated: (() -> ())?,
	PlaySfx: ((kind: "Hover" | "Press" | "Release") -> ())?,
}

local function isReducedMotion(): boolean
	local ok, val = pcall(function()
		return (GuiService :: any).ReducedMotionEnabled
	end)
	return ok and val == true
end

local function bindJuicyButton(slot: GuiObject, hitbox: GuiButton, visual: GuiObject, opts: JuicyButtonOptions): () -> ()
	local uiScale = visual:FindFirstChildOfClass("UIScale") or Instance.new("UIScale")
	uiScale.Parent = visual

	hitbox.AutoButtonColor = false
	slot.ClipsDescendants = false
	visual.AnchorPoint = Vector2.new(0.5, 0.5)
	visual.Position = UDim2.fromScale(0.5, 0.5)

	local baseZ = slot.ZIndex
	local hoverScale = opts.HoverScale or 1.07
	local pressScale = opts.PressScale or 0.93
	local hovered, pressed = false, false
	local activeInput: InputObject? = nil

	local function refresh(popOvershoot: number?)
		local reduced = isReducedMotion()
		slot.ZIndex = if hovered or pressed then baseZ + 10 else baseZ
		if reduced then
			spr.stop(uiScale, "Scale")
			uiScale.Scale = 1
			return
		end
		if popOvershoot then
			uiScale.Scale = popOvershoot
		end
		local target = if pressed then pressScale elseif hovered then hoverScale else 1
		local damping = if pressed then 0.85 else 0.52
		local freq = if pressed then 28 else 22
		spr.target(uiScale, damping, freq, { Scale = target })
	end

	local conns: { RBXScriptConnection } = {}
	table.insert(conns, hitbox.MouseEnter:Connect(function()
		if UserInputService:GetLastInputType() == Enum.UserInputType.Touch then
			return
		end
		hovered = true
		if opts.PlaySfx and not pressed then
			opts.PlaySfx("Hover")
		end
		refresh()
	end))
	table.insert(conns, hitbox.MouseLeave:Connect(function()
		hovered = false
		pressed = false
		refresh()
	end))
	table.insert(conns, hitbox.SelectionGained:Connect(function()
		hovered = true
		if opts.PlaySfx then
			opts.PlaySfx("Hover")
		end
		refresh()
	end))
	table.insert(conns, hitbox.SelectionLost:Connect(function()
		hovered = false
		pressed = false
		refresh()
	end))
	table.insert(conns, hitbox.InputBegan:Connect(function(input)
		local t = input.UserInputType
		if t == Enum.UserInputType.MouseButton1 or t == Enum.UserInputType.Touch
			or input.KeyCode == Enum.KeyCode.Return or input.KeyCode == Enum.KeyCode.ButtonA then
			pressed = true
			activeInput = input
			if opts.PlaySfx then
				opts.PlaySfx("Press")
			end
			refresh()
		end
	end))
	table.insert(conns, UserInputService.InputEnded:Connect(function(input)
		if pressed and input == activeInput then
			pressed = false
			activeInput = nil
			refresh()
		end
	end))
	table.insert(conns, hitbox.Activated:Connect(function()
		pressed = false
		activeInput = nil
		if opts.PlaySfx then
			opts.PlaySfx("Release")
		end
		refresh(1.10) -- Immediate juicy velocity impulse on release!
		if opts.OnActivated then
			opts.OnActivated()
		end
	end))

	return function()
		for _, c in conns do
			c:Disconnect()
		end
		spr.stop(uiScale)
		if slot.Parent then
			slot.ZIndex = baseZ
		end
	end
end
```

---

## 3. Animated `UIGradient` Shimmers Without Corner Bleed

- `ClipsDescendants = true` always clips to an axis-aligned rectangle and cuts through `UICorner` radii if a child rotates.
- Instead: keep `ShimmerOverlay` (`Frame`) unrotated (`Rotation = 0`) with an identical `UICorner`, set `BackgroundTransparency = 0`, and put the angle (`Rotation = 25`) + transparency curve (`1 -> 1 -> 0.25 -> 1 -> 1`) on its child `UIGradient`.
- Sweep `UIGradient.Offset` across `[-1.35, 1.35]` in a **single shared `RunService.PreRender` loop**, skipping computation whenever the target or an ancestor `LayerCollector.Enabled` / `GuiObject.Visible` is `false` or `ReducedMotionEnabled` is `true`.

---

## 4. `CanvasGroup` Isolation, Traps & Fade-Swap Pattern

`CanvasGroup` renders its entire subtree into an offscreen GPU render target so `GroupTransparency`, `GroupColor3`, and rounded `UICorner` clipping affect nested images and text uniformly.

### Critical Guardrails
1. **`ZIndexBehavior = Enum.ZIndexBehavior.Sibling` required:** Under `Global`, ancestor-child ordering is undefined and `CanvasGroup` stops rendering properly.
2. **`ClipsDescendants` is always `true`:** Inset inner modal cards (`~16–24px`) from the `CanvasGroup` boundary so outer `UIStroke`s and `UIShadow`s are not sliced off.
3. **Never host continuously animated elements inside a resting `CanvasGroup`:** Even moving a `UIGradient.Offset` shimmer or ticking timer label inside a visible `CanvasGroup` triggers **full offscreen re-rasterization every frame**, plus low graphics quality settings (Client level <= 3) aggressively downscale and blur `CanvasGroup` text/icons, and sizes > `2048px` lose sharpness.
   - **Recommended pattern:** Only parent modal content under the `CanvasGroup` during the short (`~0.18–0.25s`) open/close `GroupTransparency` fade (set `ModalCanvas.Interactable = false` immediately when closing starts; set `ModalCanvas.Visible = false` when closed, or temporarily hold interactive shimmer cards in a sibling frame once `GroupTransparency == 0`).

---

## 5. Layered Motion, Staggered Weight & Clean UI SFX

- **Outer-travel + Inner-accent layering:** Animate a parent container for modal slide/scale reveal while an inner clipped frame plays secondary sparkles/gleams; delay shadow or secondary info frames by `~0.05–0.10s` for physical weight.
- **Grid card cascade:** Stagger entry delays on visible cards by `math.min(index * 0.035, 0.28)` so long lists pop crisply rather than blocking interaction.
- **UI SFX polish:** Preload UI sounds with `ContentProvider:PreloadAsync`, assign a dedicated `SoundGroup`, apply `~0.035s` cooldown on `Hover` sounds so fast cursor sweeps do not buzz, apply subtle `±3–5%` `PlaybackSpeed` randomization, and clone transient sounds (destroyed on `Ended`/`Stopped`) so rapid clicks never truncate sound tails.
