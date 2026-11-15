---
name: roblox-ui-3d-stage
description: Use when building Roblox 3D UI, angled HUDs, camera-pinned SurfaceGui menus, Screen3D interfaces, mouse-parallax title cameras, multi-view tab camera switching with DepthOfFieldEffect or BlurEffect, animated NPC/character menu scenes, or ViewportFrame + WorldModel 3D preview cards.
---

# Roblox UI 3D Stage

Combine 3D perspective UI, parallax cameras, depth-of-field blur, and animated menu scenes while preserving touch/gamepad navigation, `WorldModel` frame budgets, and Play-state cleanup. Read `roblox-camera` first for core `Scriptable` & `PreRender` ownership; read [references/full.md](references/full.md) for full camera/NPC/viewport rigs.

## Non-Negotiable Rules

1. **Preserve native UI input in 3D menus.**
   - Use **`Screen3D`** (`Screen3D.new(screenGui, distance)` -> `GetComponent3D(obj)`) or camera-pinned **`SurfaceGui` in `PlayerGui` (`Adornee = part`)**, never `SurfaceGui` in `Workspace`. `PlayerGui` routing preserves native `GuiButton` clicks, hovers, scrolling, and Gamepad `SelectedObject` navigation without custom `Workspace:Raycast` hacks.
   - Set `LightInfluence = 0`, `Brightness` (`~0.85–1.0`) so `BloomEffect` never washes out labels, and `CanQuery = true` **only** on active `Adornee` parts (`false` on hidden panels and camera markers).
2. **Never leave hidden `WorldModel`s mounted.** Inside `ViewportFrame`, `WorldModel` **still steps physics and updates while invisible**. Use alpha-bled `512x512` PNGs for dense grids, mount `ViewportFrame` + `WorldModel` lazily for the active tab only, orbit `CurrentCamera` instead of pivoting rigged parts, clear `viewport.CurrentCamera = nil` before teardown, and destroy inactive `WorldModel`s on tab exit or Play.
3. **Protect 3D UI from `DepthOfFieldEffect` near-blur.** `DepthOfFieldEffect` blurs outside `[FocusDistance - InFocusRadius, FocusDistance + InFocusRadius]`, including 3D panels `~4–6` studs ahead of the lens. Keep `NearIntensity = 0` (or include UI distance inside the focus band), and snapshot/restore pre-existing `Lighting` effects on exit.
4. **Keep mouse parallax clamped, smoothed, and touch-safe.** Step in `RunService.PreRender` (or a unified render pass with 3D panel followers): clamp center-relative cursor delta to `[-1, 1]`, smooth exponentially (`1 - math.exp(-k * dt)`), bound stud/angle travel, skip mouse tilt on `UserInputType.Touch` or `ReducedMotionEnabled`, and guard against `CameraModule` resetting `CameraType`.
5. **Isolate menu scenes and tear down cleanly on Play.** Clone `MenuScene` from `ReplicatedStorage` (`StreamingEnabled`-safe). For staged NPCs, anchor **only** `HumanoidRootPart` (limbs unanchored `Massless = true`, `EvaluateStateMachine = false`). On Play, fade an opaque veil in, unbind render steps, `:Stop(0)`/`:Destroy()` `AnimationTrack`s, destroy cloned scene/UI, restore `CoreGui`, `Lighting`, and `PlayerModule:GetControls()`, then restore `CameraType.Custom` + `CameraSubject`.

## Modality & Trap Guide

| Goal | Pattern | Classic Trap Avoided |
| :--- | :--- | :--- |
| Angled HUD / layered 3D buttons | `Screen3D` or `PlayerGui` `SurfaceGui` (`Adornee`) | Manual raycasts break gamepad nav & blocking |
| Multi-view 3D lobby camera | Snapshot-interpolated `Cameras/*` Parts + `DepthOfFieldEffect` | Tween conflicts on rapid clicks; `NearIntensity` blurring 3D UI |
| Idle barista/mascot menu NPC | Cloned scene + anchored HRP only + `Animator` idle | Anchored limbs freeze `Motor6D`; `StreamingEnabled` unloads scene |
| 3D rotating item preview | Lazy `ViewportFrame` + `WorldModel`, orbit `CurrentCamera` | Hidden `WorldModel` drains CPU/GPU while `Enabled = false` |
