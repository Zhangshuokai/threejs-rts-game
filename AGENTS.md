# AGENTS.md

## What this repo is
Greenfield workspace for **AI-generated three.js scenes/animations** (施工场景 is the first domain, not the only one). No code, git, `package.json`, or build config yet — the only artifact is the ruleset:

`threejs-施工场景动画-提示词规则.md` ← construction domain source of truth + copy-paste prompt templates.

The rules below are the ones agents most often get wrong; they generalize from that research to **any** three.js scene. Don't re-derive them.

## General three.js generation rules (any scene)
- Lock `three` ~r185+; use `renderer.setAnimationLoop` + `Timer/Clock.getDelta()`; **never** frame-rate-dependent `+= constant`. Handle window resize.
- Scaffold predefines `scene` / `camera` / `renderer` / `controls`; do not recreate them. Output runnable JS; no Markdown in generated files.
- Renderer: try `WebGPURenderer`, fall back to `WebGLRenderer`; `setPixelRatio(Math.min(dpr, 2))`; `ACESFilmicToneMapping`; `PCFSoftShadowMap`.
- Ground = large `PlaneGeometry`; objects near the origin; component scale `0.5–5`; every mesh `castShadow = true`, ground `receiveShadow = true`.
- Sun `DirectionalLight` needs **explicit shadow-camera bounds** + 2048 map; add `HemisphereLight` + small ambient. Sky/`PMREMGenerator.fromScene` optional.
- Build in passes, never one-shot: `blockout → structure → form → material → lighting → interaction → optimization`. Spec non-trivial scenes before writing code.
- Prefer code-only geometry: primitives / `Shape` extrude / curve `tube` / displacement / canvas textures. No external model dependencies unless asked.
- Repeated parts → `InstancedMesh`. Only sync/update objects that actually moved. Use deterministic seeds for procedural noise.
- Root container `transform.scale` must stay `[1,1,1]`; hide nodes with transparency, not scale.
- Adjacent parts must physically connect/overlap `0.02–0.05` at seams (no floating parts).
- Avoid flat 2.5D slabs; verify with a distinct-Z count (planar extrusion ≈ 6–10 planes, real volume > 11).
- Describe form with 3D terms, never "nice/smooth/shiny".

## Motion / animation rules (any scene)
- Model motion as an explicit state machine; interpolate with per-axis speed-limited easing, never teleport or linear snap.
- Push driven bodies `kinematic → dynamic` when released, so gravity/contact resolves naturally.
- Fixed physics `timestep = 1/60`, cap ≤ 8 substeps/frame; count durations in **physics steps/timers, never wall-clock** (`Date.now()` breaks under variable speed).
- "有 clip ≠ 会播放": a `mixer` + action alone proves nothing — sample actual transforms at a time and compare to the track value.

## Verification (no build/test/lint exists yet)
- Playwright tooling is available — open the generated HTML and confirm it renders **and** animates.
- Capture a **turntable** (front/right/rear/left); one viewpoint is not evidence.
- Self-correction decision, one per pass: `continue | refine-spec | refine-code | request-input | stop`.

## Construction-domain layer (verified against `construction-sim`)
- Sequence: `footing → 逐层(柱 → 主梁 → 次梁 → 楼板 → 墙板 → 校验) → 屋顶 → 完工报告`.
- `主梁` along X, `次梁` along Z, **staggered 0.6** in Y so perpendicular beams never share a corner volume.
- Wall panels leave **0.1 m clearance each end**.
- Tower crane hierarchy: `base → slewing(θ) → jib → trolley(r) → hookGroup(cable.scale.y)`. Drive by polar `(θ, r, h)`.
- Per-axis speed-limited easing (`vT≈0.85, vR≈5.0, vH≈8.0`); hoist **only after** slew/radius are near settled.
- Release only when crane settled **and** horiz `< 0.07` and vert within `0.2`; then `kinematic → dynamic` drop. `SETTLE` = ~42 physics steps (60 Hz × 0.7 s).
- Component state machine: `DELIVERED → TO_PICK → ATTACH → LIFT → TRANSIT → LOWER → SETTLE → VALIDATE → FASTEN`.
- Placement verdict: `<0.08 m` OK, `0.08–0.25 m` WARN, `>0.25 m` or rot `>5°` RESEAT.
- Check support logic (柱有基础、梁端落柱、板落梁) and no interpenetration.

## Reference implementations to copy from
- General scene + animation: three.js official `webgl_animation_keyframes` (AnimationMixer, render loop, lighting, resize).
- Prompt scaffold + raw-JS output rule: `Mishi17o/ArvixAI-3D-Gen` → `src/prompts.py`, `src/renderer.py`.
- Staged/quality-gated codegen rules: `img2threejs/img2threejs` → `SKILL.md`, `grimoire/build/geometry_patterns.md`, `grimoire/readiness/animation_contract.md`.
- Construction sequence + crane + validation: `caimingye78/construction-sim` → `src/crane.js`, `src/sim.js`, `src/building.js`, `src/actors.js`, `src/main.js` (stack: `three ^0.185`, `@dimforge/rapier3d-compat`, Vite).

## Environment quirks (this machine)
- General search engines (Bing / DuckDuckGo / Brave / Ecosia) return filtered or unusable results. Research via `api.github.com` and official three.js sources.
- `raw.githubusercontent.com` is blocked; fetch file contents through the GitHub contents API (base64) instead. Bash has outbound network access.
