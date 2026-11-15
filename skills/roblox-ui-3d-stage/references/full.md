# Roblox UI 3D Stage — Full Reference

Companion guide to `roblox-ui-3d-stage`. Cross-reference `roblox-camera` for foundational `Scriptable` camera ownership and `roblox-ui-dynamic-juice` for button/modal motion polish.

---

## 1. Choosing the Right 3D UI Modality

| Modality | When to Choose | Setup & Input Behavior |
| :--- | :--- | :--- |
| **`Screen3D`** ([DevForum 3273671](https://devforum.roblox.com/t/screen3d-a-3d-ui-framework-that-just-works/3273671)) | Curved/tilted HUDs, holographic dashboards, nested 3D pop-outs | `local s3d = Screen3D.new(screenGui, distance)` -> `:GetComponent3D(guiObj)` with `.offset` `CFrame` pivoting on `AnchorPoint`; `:Enable()`/`:Disable()`. Converts `ScreenGui` layout into camera-pinned `SurfaceGui`s automatically. |
| **Camera-Pinned `SurfaceGui` (`PlayerGui`)** | Physical-thickness 3D panels (`Part` geometry + neon trim) anchored smoothly in front of the lens with lag/parallax | Keep the `SurfaceGui` **inside `PlayerGui`** (`Adornee = panelPart`). Native `TextButton`/`ImageButton` hover, press, scroll, and Gamepad selection work automatically—no custom raycast needed unless detecting clicks on non-`SurfaceGui` 3D meshes. Set `LightInfluence = 0`, `Brightness = 0.9`, `ZIndexBehavior = Sibling`. |
| **Hybrid 2D `ScreenGui` + 3D Camera Stage** | Most shops, title screens, and settings screens | Crisp sharp `TextScaled`/`UIStroke` 2D HUD (`CoreUISafeInsets`) layered over a dynamic mouse-parallax / tab-switching `Scriptable` 3D camera + `DepthOfFieldEffect`. Best mobile text crispness & gamepad compatibility. |
| **`ViewportFrame` (+ optional `WorldModel`)** | Spinning 3D item/pet cards or character inspect turntables inside 2D frames | Isolated `CurrentCamera` + local directional/ambient light. Supports `WorldModel` only when rigged animation or physics is required. |

---

## 2. Camera-Pinned 3D Panel Aspect & Orientation Math

When positioning an adorned `Part` in front of `CurrentCamera` (rotation + frustum fit across wide `16:9` desktops and ultra-wide `20:9` mobile phones):
1. Flip the panel `180°` on Y (`math.pi`) if using `Enum.NormalId.Front` so the UI faces the camera lens (`-Z` in Roblox camera space).
2. Derive half-bounds cleanly from camera `FieldOfView` and `ViewportSize`:
   - `halfH = distance * math.tan(math.rad(fov * 0.5))`
   - `aspect = viewportSize.X / math.max(viewportSize.Y, 1)`
   - `halfW = halfH * math.clamp(aspect, 0.65, 1.95)`
3. Place buttons relative to `halfW` / `halfH` (`posX = -halfW * 0.55`), and blend `baseCamCFrame * parallaxCFrame:Lerp(CFrame.identity, 0.5)` so panels trail camera parallax gently with 3D weight.

---

## 3. Mouse-Parallax & Interruptible Multi-View Camera Controller

### Key Rules
- **Clone the menu scene locally (`ReplicatedStorage.MenuAssets.MenuScene` -> `Workspace`)** or wait/pin parts cleanly: relying purely on distant unstreamed `Workspace` menu stages breaks when `Workspace.StreamingEnabled = true` unloads parts before the camera moves there.
- **Keep `CameraPoints` parts invisible & non-interactive:** `Anchored = true`, `Transparency = 1`, `CanCollide = false`, `CanTouch = false`, `CanQuery = false`.
- **Snapshot active pose on rapid view switches:** When switching `Home -> Shop -> Customize` mid-transition, capture `_currentBaseCFrame`, `_currentFov`, and `_currentDof` as the new start values rather than resetting to static part CFrames.
- **DepthOfField Focal Plane Safety:** Setting `NearIntensity = 0` guarantees camera-space 3D `SurfaceGui` elements (`~4–6` studs ahead of the lens) stay sharp while `FocusDistance` + `InFocusRadius` blur the background lobby (`FarIntensity = 0.65–0.85`). Always record pre-existing `DepthOfFieldEffect` states in `Lighting` before enabling menu DOF and restore them on exit.

```lua
--!strict
local Lighting = game:GetService("Lighting")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local GuiService = game:GetService("GuiService")
local Workspace = game:GetService("Workspace")

export type CameraViewSpec = {
	TargetPart: BasePart,
	Fov: number,
	ParallaxWeight: number,
	DofFocusDistance: number,
	DofInFocusRadius: number,
	DofFarIntensity: number,
}

local MenuCameraRig = {}
MenuCameraRig.__index = MenuCameraRig

function MenuCameraRig.new(views: { [string]: CameraViewSpec }, initialViewId: string)
	local camera = Workspace.CurrentCamera :: Camera
	local first = views[initialViewId]

	local savedDofs: { { Effect: DepthOfFieldEffect, Enabled: boolean } } = {}
	for _, child in Lighting:GetChildren() do
		if child:IsA("DepthOfFieldEffect") then
			table.insert(savedDofs, { Effect = child, Enabled = child.Enabled })
			child.Enabled = false
		end
	end

	local dof = Instance.new("DepthOfFieldEffect")
	dof.Name = "MenuStageDOF"
	dof.NearIntensity = 0 -- Protects camera-pinned 3D UI from near-plane blur
	dof.FocusDistance = first.DofFocusDistance
	dof.InFocusRadius = first.DofInFocusRadius
	dof.FarIntensity = first.DofFarIntensity
	dof.Enabled = true
	dof.Parent = Lighting

	local self = setmetatable({
		_camera = camera,
		_views = views,
		_activeId = initialViewId,
		_prevType = camera.CameraType,
		_prevFov = camera.FieldOfView,
		_dof = dof,
		_savedDofs = savedDofs,
		_startCF = first.TargetPart.CFrame,
		_currCF = first.TargetPart.CFrame,
		_goalCF = first.TargetPart.CFrame,
		_startFov = first.Fov,
		_currFov = first.Fov,
		_goalFov = first.Fov,
		_startBlur = first.DofFarIntensity,
		_currBlur = first.DofFarIntensity,
		_goalBlur = first.DofFarIntensity,
		_startFocus = first.DofFocusDistance,
		_currFocus = first.DofFocusDistance,
		_goalFocus = first.DofFocusDistance,
		_elapsed = 1,
		_duration = 0.7,
		_mouseX = 0,
		_mouseY = 0,
		_destroyed = false,
	}, MenuCameraRig)

	camera.CameraType = Enum.CameraType.Scriptable
	self._typeConn = camera:GetPropertyChangedSignal("CameraType"):Connect(function()
		if not self._destroyed and camera.CameraType ~= Enum.CameraType.Scriptable then
			camera.CameraType = Enum.CameraType.Scriptable
		end
	end)

	return self
end

function MenuCameraRig:SetView(viewId: string)
	local spec = self._views[viewId]
	if not spec or self._destroyed then
		return
	end
	self._activeId = viewId
	self._startCF, self._goalCF = self._currCF, spec.TargetPart.CFrame
	self._startFov, self._goalFov = self._currFov, spec.Fov
	self._startBlur, self._goalBlur = self._currBlur, spec.DofFarIntensity
	self._startFocus, self._goalFocus = self._currFocus, spec.DofFocusDistance
	self._elapsed = 0
end

function MenuCameraRig:Step(dt: number)
	if self._destroyed then
		return
	end
	self._elapsed = math.min(self._elapsed + dt, self._duration)
	local alpha = TweenService:GetValue(self._elapsed / self._duration, Enum.EasingStyle.Quint, Enum.EasingDirection.Out)

	self._currCF = self._startCF:Lerp(self._goalCF, alpha)
	self._currFov = self._startFov + (self._goalFov - self._startFov) * alpha
	self._currBlur = self._startBlur + (self._goalBlur - self._startBlur) * alpha
	self._currFocus = self._startFocus + (self._goalFocus - self._startFocus) * alpha

	self._dof.FarIntensity = self._currBlur
	self._dof.FocusDistance = self._currFocus

	local spec = self._views[self._activeId]
	local reducedOk, reducedVal = pcall(function()
		return (GuiService :: any).ReducedMotionEnabled
	end)
	local disableParallax = (reducedOk and reducedVal == true)
		or not UserInputService.MouseEnabled
		or UserInputService:GetLastInputType() == Enum.UserInputType.Touch

	local goalX, goalY = 0, 0
	if not disableParallax then
		local vp = self._camera.ViewportSize
		if vp.X > 0 and vp.Y > 0 then
			local m = UserInputService:GetMouseLocation()
			goalX = math.clamp((m.X / vp.X - 0.5) * 2, -1, 1)
			goalY = math.clamp((m.Y / vp.Y - 0.5) * 2, -1, 1)
		end
	end

	local smooth = 1 - math.exp(-6.5 * dt)
	self._mouseX += (goalX - self._mouseX) * smooth
	self._mouseY += (goalY - self._mouseY) * smooth

	local weight = if spec then spec.ParallaxWeight else 1
	local offsetCF = CFrame.new(self._mouseX * 0.30 * weight, -self._mouseY * 0.16 * weight, 0)
		* CFrame.Angles(-self._mouseY * math.rad(1.4) * weight, -self._mouseX * math.rad(2.2) * weight, 0)

	self._camera.CFrame = self._currCF * offsetCF
	self._camera.FieldOfView = self._currFov
end

function MenuCameraRig:Destroy()
	if self._destroyed then
		return
	end
	self._destroyed = true
	if self._typeConn then
		self._typeConn:Disconnect()
	end
	self._dof:Destroy()
	for _, s in self._savedDofs do
		if s.Effect.Parent then
			s.Effect.Enabled = s.Enabled
		end
	end
	self._camera.FieldOfView = self._prevFov
	self._camera.CameraType = if self._prevType == Enum.CameraType.Scriptable
		then Enum.CameraType.Custom
		else self._prevType
end
```

---

## 4. Staged Menu NPCs & Play-State Orchestrator

Following modular title-screen architecture:
1. **Split concerns cleanly:** Orchestrator signals (`MenuOpen:Fire(true/false)`) coordinate four focused client controllers: (1) Camera + DOF rig, (2) Button/HUD juice, (3) Staged Character `Animator`, (4) Transition veil + Play handoff.
2. **Staged NPC rig rules:**
   - Anchor **only** `HumanoidRootPart` (leave limbs unanchored with `CanCollide = false`, `Massless = true` so `Motor6D` joints animate cleanly without falling).
   - Set `Humanoid.EvaluateStateMachine = false`, `DisplayDistanceType = Enum.HumanoidDisplayDistanceType.None`, `HealthDisplayType = Enum.HumanoidHealthDisplayType.AlwaysOff`.
   - Load idle `Animation` tracks via `Humanoid.Animator` (`Priority = Enum.AnimationPriority.Idle`, `Looped = true`); call `track:Stop(0)` and `track:Destroy()` immediately when Play starts.
3. **Transition-Veil Play Handoff:** Fade a full-screen overlay (`ScreenInsets = None`) to opaque -> `RunService:UnbindFromRenderStep(...)` -> destroy menu UI/NPCs/`WorldModel`s/local `MenuScene` -> re-enable `CoreGui` and `PlayerModule:GetControls()` -> restore `CameraType.Custom` & `CameraSubject = humanoid` -> fade overlay back out.

---

## 5. `ViewportFrame` + `WorldModel` Performance Rules

- **`WorldModel` keeps ticking when invisible:** Roblox still steps physics and animations inside a `WorldModel` even if its parent `ViewportFrame` or `ScreenGui` has `Visible = false` / `Enabled = false`.
- **Lazy Mount / Unmount:** Instantiate preview `ViewportFrame` + `WorldModel` contents **only for the currently active category/inspector**, and destroy or unparent the `WorldModel` when leaving the tab or closing the menu.
- **Dense shop grids (20+ cards):** Use pre-rendered `512x512` alpha-bled PNG portraits in grid tiles and reserve a single live `ViewportFrame` + `WorldModel` for the selected Inspector spotlight.
- **Turntable optimization:** Auto-frame via `model:GetBoundingBox()`, keep static/rigged parts anchored cleanly at origin, and **orbit `ViewportFrame.CurrentCamera`** around `(0, 0, 0)` instead of calling `PivotTo` on multi-part models every frame. Clear `viewport.CurrentCamera = nil` before cleanup so cached `Camera` instances do not leak.
- **Lighting caveat:** `ViewportFrame` does not render workspace shadows or `Lighting` post-processing (`Bloom`/`DOF`); configure each `ViewportFrame`'s own `Ambient`, `LightColor`, and `LightDirection`.
