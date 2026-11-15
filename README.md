# roblox-ui-skills

Three focused agent skills for modern Roblox UI visual craft, dynamic motion/juice, and 3D camera-coupled menu scenes.

## Included Skills

1. **[`roblox-ui-style-craft`](skills/roblox-ui-style-craft/SKILL.md)**  
   Visual craft & scaling: Figma `Frame` workflow & Alpha-Bleed (`pixelfix`) PNG prep, cartoony 3D button extrusions & shop hierarchy, Roblox Stud UI (`ScaleType.Tile` + `TileSize`), native 2026 styling APIs (`UIShadow`, per-corner `UICorner`, `UIGradient` `Type`/`Enum.GradientTileMode`, `UIStroke.StrokeSizingMode.ScaledSize` + `BorderOffset`, `StyleSheet`/`StyleRule`/`StyleQuery`), and responsive Device Emulator scaling.

2. **[`roblox-ui-dynamic-juice`](skills/roblox-ui-dynamic-juice/SKILL.md)**  
   Dynamic 2D motion & polish: layout-safe `Slot -> Hitbox -> Visual + UIScale` interactions, choosing between `StyleRule:SetPropertyTransition`, Fraktality's `spr` springs, and shared `RunService.PreRender` `UIGradient.Offset` sweeps, `CanvasGroup` isolation & low-graphics traps, layered container motion, UI SFX cooldown/pitch jitter, and `ReducedMotionEnabled` accessibility.

3. **[`roblox-ui-3d-stage`](skills/roblox-ui-3d-stage/SKILL.md)**  
   3D UI, parallax cameras & living menu scenes: `Screen3D` vs `PlayerGui`-parented camera-pinned `SurfaceGui` (`Adornee` input preservation), clamped exponential mouse parallax, `DepthOfFieldEffect` `NearIntensity = 0` focal protection, client-cloned `MenuScene` (`StreamingEnabled`-safe) with root-anchored NPC `Animator`, lazy `ViewportFrame` + `WorldModel` previews, and clean Play-state teardown.

## Installation

Copy any folder under `skills/` into your agent skills directory (e.g., `~/.gemini/config/skills/` or `<repo>/.agents/skills/`), or install via `npx skills add Kitkopika/roblox-ui-skills`.
