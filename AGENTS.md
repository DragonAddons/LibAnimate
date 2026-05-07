# LibAnimate AGENTS.md

## Core Design

- **OnUpdate-based rendering** - single shared driver frame; avoids WoW's buggy Animation/AnimationGroup system.
- **Keyframe interpolation** - animations are keyframe lists with property values at progress points.
- **Per-segment easing** - named presets plus full cubic-bezier via Newton-Raphson solver.
- **`RegisterAnimation` API** - external addons can register custom animations.
- **State tables** - `lib.animations`, `lib.activeAnimations`, `lib.animationQueues`.
- Private functions stay local; public API methods attach to the `lib` table and take `self`.
- LuaLS annotations are used extensively for public types and APIs.
- Luacheck: ignore code 212 for `self` parameters.

## Supported WoW Versions

Retail (110207 / 120001), TBC Anniversary (20505), MoP Classic (50502 / 50503).

## Animation Definition Format

```lua
{
    type = "entrance",        -- "entrance", "exit", or "attention"
    defaultDuration = 0.6,    -- seconds
    defaultDistance = 300,     -- pixels
    keyframes = {
        { time = 0.0, translateX = 0, translateY = 1.0, scale = 0.7, alpha = 0.7 },
        { time = 0.8, translateY = 0, scale = 0.7, alpha = 0.7 },
        { time = 1.0, scale = 1.0, alpha = 1.0 },
    },
}
```

**Properties:** `translateX`, `translateY` (fraction of distance), `scale` (uniform), `alpha` (opacity).
**Defaults:** translateX=0, translateY=0, scale=1.0, alpha=1.0 (`PROPERTY_DEFAULTS` table).
**Coordinates:** WoW system (positive Y = up, positive X = right).

## Driver Error Handling

The shared OnUpdate driver must never crash. Two rules:

- Use `error("message", 2)` (level 2) for public-API input validation so the report points at the caller.
- Wrap user callbacks with `pcall()` + `geterrorhandler()()`. An error in one animation must not kill the driver loop or other active animations.

## Packaging

`.pkgmeta`:
- `package-as`: LibAnimate
- `enable-toc-creation`: yes
- External dependency: LibStub (from wowace repos)

## Pitfalls

- The OnUpdate driver runs every frame for every active animation - keep per-tick work allocation-free; do not create tables or closures inside the interpolation path.
- Frame recycled while an animation is active: callers must call the lib's stop API before releasing/repooling, or stale entries linger in `lib.activeAnimations`.
- Cubic-bezier easing solves `t` numerically; supply only well-formed control points (x in [0, 1]) or the Newton-Raphson loop misbehaves.
