# AGENTS.md

## What this repo is
Workspace for **AI-generated three.js scenes/animations** (施工场景 was the first domain, not the only one). No git or build config — artifacts:

- `threejs-施工场景动画-提示词规则.md` ← construction domain source of truth + copy-paste prompt templates.
- `threejs-场景动画-提示词模板与编写技能.md` ← generalized 10-section prompt structure (`+模拟契约/验收接口`), the `R1–R10` rules for **real-time/emergent** scenes (crowd combat, crowds, traffic), WebGPU/WebGL r185 pitfalls, the verification recipe, and measured perf budgets. **Read this before writing a prompt for anything that is not a fixed procedure.**
- `civil-war-battle.html` ← worked reference: 南北战争 group combat (three r185, ~470 instanced entities, emergent outcome, instanced billboard VFX + bloom/grade post-processing, single file, `?fx=0` disables post-fx, `?pause=1` for deterministic stepping).

The rules below are the ones agents most often get wrong; they generalize from that research to **any** three.js scene. Don't re-derive them.

## General three.js generation rules (any scene)
- Lock `three` ~r185+; use `renderer.setAnimationLoop` + `Timer/Clock.getDelta()`; **never** frame-rate-dependent `+= constant`. Handle window resize.
- Scaffold predefines `scene` / `camera` / `renderer` / `controls`; do not recreate them. Output runnable JS; no Markdown in generated files.
- Renderer: try `WebGPURenderer`, fall back to `WebGLRenderer`; `setPixelRatio(Math.min(dpr, 2))`; `ACESFilmicToneMapping`; `PCFShadowMap` (**`PCFSoftShadowMap` is deprecated in r185**).
  The WebGPU build does **not** export `WebGLRenderer` and rejects `ShaderMaterial`/GLSL — pick the build at load time (inject the importmap from a boot script after probing `navigator.gpu`, on `DOMContentLoaded`), and use canvas textures + standard materials only.
- Ground = large `PlaneGeometry`; **object** size `0.5–5 m` (large scene extent is fine — spread by coordinates, never by root scale); every mesh `castShadow = true`, ground `receiveShadow = true`.
- Sun `DirectionalLight` needs **explicit shadow-camera bounds** sized to the *max activity radius* (not the initial layout) + 2048 map; add `HemisphereLight` + small ambient. Shadow texel = `2×halfExtent/mapSize`, so sub-0.2 m props get no usable shadow at ±170/2048.
- Build in passes, never one-shot: `blockout → structure → form → material → lighting → interaction → optimization`. Spec non-trivial scenes before writing code.
- Prefer code-only geometry: primitives / `Shape` extrude / curve `tube` / displacement / canvas textures. No external model dependencies unless asked.
- Repeated parts → `InstancedMesh`. Only sync/update objects that actually moved. Use deterministic seeds for procedural noise — and **separate the gameplay PRNG from the visual-effects PRNG**, or retuning smoke will change the outcome.
- Root container `transform.scale` must stay `[1,1,1]`; hide nodes with transparency, not scale (scale-0 **instance matrices** are the correct way to hide individual instances).
- Adjacent parts must physically connect/overlap `0.02–0.05` at seams (no floating parts).
- Avoid flat 2.5D slabs; verify with a distinct-Z count (planar extrusion ≈ 6–10 planes, real volume > 11).
- Describe form with 3D terms, never "nice/smooth/shiny".

## Motion / animation rules (any scene)
- Model motion as an explicit state machine; interpolate with per-axis speed-limited easing, never teleport or linear snap.
- Push driven bodies `kinematic → dynamic` when released, so gravity/contact resolves naturally.
- Fixed physics `timestep = 1/60`, cap ≤ 8 substeps/frame; count durations in **physics steps/timers, never wall-clock** (`Date.now()` breaks under variable speed).
- "有 clip ≠ 会播放": a `mixer` + action alone proves nothing — sample actual transforms at a time and compare to the track value.
- **Tracking a moving target** (formation slot, hook, attach point): the stop dead-zone must be **smaller than the target's per-frame displacement** (`v·dt`). A dead-zone of 0.22 m against 2.8 m/s @ 60 Hz (0.047 m/frame) makes agents stutter (stop 1 frame in 4); if the speed cap is also ≤ target speed they fall behind permanently. Use ~0.02 m.
- **Never milestone on an absolute coordinate** (`Math.abs(cz) > 6` is already true at spawn). Use a signed/directional form (`cz * toward > 6`) or a distance.
- **Morale: accumulate-and-decay, with dual rout thresholds** — permanent shock (e.g. from ranged fire) makes units break the moment melee starts. `morale = 1 + resolve − 1.65×lossRatio − shock`, `shock` capped and decaying, rout on `men<30% || morale<0.22`.
- **Effects budget = radius × lifetime × opacity**, not count. Smoke ≤6 m / ≤2.5 s / ≤0.5 opacity; a paused sim freezes every effect at peak, so screenshots exaggerate long lifetimes.

## Effects layer (verified in `civil-war-battle.html`)
- **Never use `THREE.Sprite` for particle effects** — sprites do not batch (~1.1 draw calls each; 333 live smoke puffs = 705 calls total). Use one `InstancedMesh` of `PlaneGeometry(1,1)` per texture/blending group, billboarded each frame with `camera.quaternion * spinQuat`, `frustumCulled = false`, `instanceMatrix.setUsage(DynamicDrawUsage)`. 5 pools (flash/smoke/dust/boom/mote) = 5 draw calls total.
- Per-instance fade: `instanceColor` scales rgb (`setColorAt`), which only works for additive pools. For **normal-blend** smoke/dust use premultiplied `CustomBlending` (`blendSrc: OneFactor`, `blendDst: OneMinusSrcAlphaFactor`) and set `instanceColor = vec3(a)`.
- Hide a dead instance with a **scale-0 matrix** and re-write its matrix only once (track `drawn`), not by `visible = false`.
- **Bloom needs HDR emissive, not a low threshold.** A daylight scene sits at linear ~0.7–1.3, so `threshold 0.82` blooms the whole frame into fog. Use `threshold ≈ 1.3` plus `instanceColor = a × 2.2–2.4` on additive pools so only muzzle flashes and shell bursts cross it. Additive pools are `toneMapped` by `OutputPass`, so >1 values still resolve to white rather than clipping.
- Post-processing (`EffectComposer` + `RenderPass` + `UnrealBloomPass` + grade `ShaderPass` + `OutputPass`) is **WebGL-only**: the WebGPU build rejects `ShaderMaterial`/GLSL, so `await import(...)` the passes inside `try/catch` behind `MODE === 'webgl'` and fall back to `renderer.render`. Composer path costs ~10–20 % fps on the software rasterizer (`?fx=0` disables it).
- Wispy smoke beats a radial gradient: build the puff as `ImageData` = value-noise fBm × a soft radial mask (`puffTex()`), or every puff reads as an identical bubble.
- With `autoReset = false` on `renderer.info` and one `info.reset()` per frame, `info()` reports the **whole frame** including every composer pass and the shadow pass — the only numbers comparable across configurations.
- **Prove PRNG separation, don't assume it.** All effect randomness (pool spin/rotation, spread, crater size, hoof dust) reads a second stream `vrnd = mulberry32(0x9D3C71)`; gameplay keeps `rnd`. Test: patch a visual constant into a copy of the file and step both to the same `t` with `?pause=1`; `state()` must be **byte-identical**. Doing this caught a real leak — raising pool capacity changed how often `emit()` found a free slot, which changed how many `rnd()` draws happened, which changed the battle. Splitting the streams changed the outcome once (union still wins, ~41.8 s); after that, effect retunes are free.

## Verification (no build/test/lint exists yet)
- Playwright tooling is available — open the generated HTML and confirm it renders **and** animates.
- Capture a **turntable** (front/right/rear/left); one viewpoint is not evidence.
- Self-correction decision, one per pass: `continue | refine-spec | refine-code | request-input | stop`.
- **Expose a debug API** in the artifact (`sim(seconds)`, `state()`, `sample(entity, part)`, `freecam(...)`, `info()`), then: step deterministically to a moment and screenshot; print a **phase timeline** (`t/phase/strength`); read **instance matrices** to prove poses animate; re-load twice and confirm `state()` is identical (determinism); record `info()` draw calls/triangles as a budget baseline.
- `?pause=1` starts paused so `sim()` steps from `t=0` — without it the render loop advances the sim before you can pause, and two "re-loads" are not comparable.
- Effects-debug surface worth keeping: `particles()` (per-pool alive), `pools(show)` (solo/mute pools by name), `blend(id, mode)`, `postfx('bloom'|'threshold'|'radius'|'grain'|'vignette', v)`, `fx()` / `fx(false)` for A/B. A silently square-looking billboard is almost always a blend/alpha problem; solo each pool instead of guessing.
- Measured budget for ~470 instanced entities (software rasterizer, 1400×820, shadow pass included): 356–382 draw calls / 3.6×10⁵ tris, effects on. **Before**: 260 visible sprites = 646 calls; 333 sprites = 705 calls. **After** (instanced billboards): 320–551 live particles = 382 calls, `?fx=0` ≈ direct render. fps is noisy on this rasterizer (18–28 fps at that resolution for both paths, composer ≈ 10–20 % slower); trust draw calls/triangles, not single fps samples.

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
