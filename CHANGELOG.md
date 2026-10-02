# Changelog

All changes are relative to upstream [`OUdefie17/Photon-GAMS`](https://github.com/OUdefie17/Photon-GAMS)
at `main` (last upstream push 2025-03-30), which is based on
[`sw-52/gluon`](https://github.com/sw-52/gluon) and [`sixthsurge/photon`](https://github.com/sixthsurge/photon).

The raw diff of the current release is kept in
[`docs/t12-code-changes.patch`](docs/t12-code-changes.patch); it applies with
`git apply -p1` (or `patch -p1`) from the repository root.

---

## t12 — 2026-09-10

First public release of this fork. **20 files, +257 / −81 lines.** The patch was verified by
re-applying it to upstream `main`: the result is byte-for-byte identical to the tested `t12` build.

### Added

* `TRANSLUCENT_ALPHA` — opacity multiplier (default `0.75`) for translucent surfaces, applied in
  `PREMULTIPLIED_ALPHA` set, and never applied to lightning (material `102`) or nether portals (`62`).
* `PARTICLE_OCCLUSION` — default `ON`; rejects particles hidden behind solid geometry, because the
  translucent layer they are drawn into is composited without a depth test.
* `PUDDLE_MODE` — `PUDDLE_MODE_PATCHY` (original random puddles) / `PUDDLE_MODE_FULL` (default: the
  whole wet, flat, outdoor ground).
* `USE_SEPARATE_ENTITY_DRAWS` — exposes Iris' `separateEntityDraws` together with the
  `program.*/gbuffers_*_translucent.enabled` table. The pack documents that these programs only
  exist on **Iris 26.1+** and that enabling them on older versions renders translucent entities and
  blocks incorrectly.
* `DITHERED_TRANSLUCENCY_FALLBACK` — default `ON`; dithered transparency for objects that should be
  translucent while separate entity draws are off.
* `en_US` / `zh_CN` option names, tooltips and enum values for all of the above.
* `docs/t12-code-changes.patch` in this repository.

### Fixed

* **Transparent textures and particles on 1.21+.** Translucent programs wrote non-premultiplied
  colour into a pipeline that blends with `ONE / ONE_MINUS_SRC_ALPHA`. They now output premultiplied
  alpha (`PREMULTIPLIED_ALPHA`); the legacy `fragment_color.a = sqrt(a)` and
  `rgb / max(a, eps)` pair was removed; blend functions for `gbuffers_textured`,
  `gbuffers_particles`, `gbuffers_particles_translucent` and `gbuffers_weather` were changed to
  `ONE ONE_MINUS_SRC_ALPHA`; rain and snow are premultiplied in `gbuffers_weather.fsh`.
* **Particle depth.** Particles are drawn through the translucent program instead of the deferred
  gbuffer (`world*/gbuffers_particles.{vsh,fsh}` now include the `*_translucent` programs). Because
  that layer has no depth test at composite time, hidden particles are discarded in-shader, and
  `gbuffers_hand` writes `colortex13 = 0` where it passes the depth test, so particles (and water)
  no longer show through the held item.
* **Particle glow.** The translucent vertex stage now assigns the particle material masks —
  `27` (particle) and `47` (glowing particle) — so glowing particles emit instead of being shaded as
  an unclassified material.
* **Directional lightmaps in translucent/particle programs.**
  `get_directional_lightmaps()` takes the scene position as a parameter instead of reading the global
  `scene_pos`; the screen-space derivatives were computed from the wrong value there, which showed up
  as wrong lightmaps/glow on particles and translucent surfaces.
* **Blanket removal of bluish particles (mod compatibility).** The hardcoded
  `if (base_color.r < 0.29 && base_color.g < 0.45 && base_color.b > 0.75) discard;` rule — meant to
  hide rain-splash sprites — deleted every bluish particle in the game, including Kaleidoscope Tavern
  tap/dishes particles and animated block models. It is now gated behind
  `rainStrength > 0.05 && light_levels.y > 0.1`, i.e. only while it is actually raining under open sky.
* **Puddles under overhangs.** The "indoors" test changed from `pow5(light_levels.y)` to
  `linear_step(14.0 / 15.0, 1.0, light_levels.y)`, and puddle surfaces are excluded from leaves
  (material `5`).
* Flickering/unstable puddle shading: `puddle_f0` 0.02 → 0.2, `puddle_roughness` 0.002 → 0.008 and
  the roughness blend uses `smoothstep(eps, 0.001, puddle)`; the flat-normal test is now
  `step(0.99, flat_normal.y)`; `update_rain_puddles()` receives `material_mask`.
* Alpha tests use the `alphaTestRef` uniform instead of a hardcoded `0.1`.

### Changed

* `IS_IRIS` compile-time branching in `gbuffers_damagedblock.fsh` and `d4_deferred_shading.fsh`
  replaced with `USE_SEPARATE_ENTITY_DRAWS`, so the branch follows the feature actually in use
  instead of the loader; the new macro is undefined on OptiFine (`#ifndef IS_IRIS
  #undef USE_SEPARATE_ENTITY_DRAWS`).
* `particles.ordering = mixed` pinned in `shaders.properties` (an earlier attempt used `before`,
  which made smoke pass through the hand as well).
* The `alphaTest.*` block in `shaders.properties` is commented out instead of forcing `off` for
  every program.

### Files

| File | +/− | Why |
| --- | --- | --- |
| `shaders/shaders.properties` | +79 / −29 | new option rows, particle program enablement, separate-entity-draws program table, blend functions, `particles.ordering`, `alphaTest.*` commented out |
| `shaders/program/gbuffers_all_translucent.fsh` | +52 / −15 | premultiplied alpha, particle occlusion, particle material path, `TRANSLUCENT_ALPHA`, rain-only splash filter, normal from derivatives before the alpha discard |
| `shaders/program/gbuffers_all_solid.fsh` | +37 / −6 | hand clears `colortex13`, `alphaTestRef`, dithered translucency fallback, rain-only splash filter |
| `shaders/settings.glsl` | +25 / −0 | `PUDDLE_MODE`, `TRANSLUCENT_ALPHA`, `PARTICLE_OCCLUSION`, `USE_SEPARATE_ENTITY_DRAWS`, `DITHERED_TRANSLUCENCY_FALLBACK` |
| `shaders/include/misc/rain_puddles.glsl` | +20 / −9 | puddle modes, parameters, leaves excluded, indoor threshold |
| `shaders/lang/zh_CN.lang` | +11 / −1 | Chinese option names/tooltips |
| `shaders/lang/en_US.lang` | +8 / −0 | English option names/tooltips |
| `shaders/program/d4_deferred_shading.fsh` | +6 / −6 | `USE_SEPARATE_ENTITY_DRAWS` gating, pass `material_mask` to puddles |
| `shaders/program/gbuffers_all_translucent.vsh` | +6 / −1 | particle material masks (`27` / `47`) |
| `shaders/program/gbuffers_damagedblock.fsh` | +2 / −3 | `USE_SEPARATE_ENTITY_DRAWS` gating |
| `shaders/include/lighting/directional_lightmaps.glsl` | +2 / −2 | explicit scene position parameter |
| `shaders/program/gbuffers_weather.fsh` | +1 / −1 | premultiplied rain/snow colour |
| `shaders/world0/gbuffers_particles.fsh` | +1 / −1 | include the translucent program |
| `shaders/world0/gbuffers_particles.vsh` | +1 / −1 | include the translucent program |
| `shaders/world1/gbuffers_particles.fsh` | +1 / −1 | include the translucent program |
| `shaders/world1/gbuffers_particles.vsh` | +1 / −1 | include the translucent program |
| `shaders/world-1/gbuffers_particles.fsh` | +1 / −1 | include the translucent program |
| `shaders/world-1/gbuffers_particles.vsh` | +1 / −1 | include the translucent program |
| `shaders/world_moon/gbuffers_particles.fsh` | +1 / −1 | include the translucent program |
| `shaders/world_moon/gbuffers_particles.vsh` | +1 / −1 | include the translucent program |

### Known limitations

* The rain-splash filter judges by colour and environment only; a tap drip that runs outdoors while
  it is raining is still removed.
* `PARTICLE_OCCLUSION = ON` also hides effects that mods intentionally draw without depth testing
  (Ars Nouveau ritual helix effects). Set it to `OFF` for that look.

---

## Development history (pre-release iterations)

These builds were made while developing the fix set; they are not published as separate releases,
but they explain how t12 reached its current form.

| Build | Result |
| --- | --- |
| t5 | Particles (embers/smoke) routed back into the pack's own particle program, fixing ember occlusion and smoke pass-through. This introduced the blanket bluish-particle discard that t8/t9 later refined. |
| t6, t7 | Experiments that forced lightning to pure white to locate a red-tinted lightning beam. The tint was not produced by the channels that were patched, so the experiment was dropped; the fix set continues from t5. |
| t8 | Removed the hardcoded bluish-particle discard. Kaleidoscope Tavern tap-water droplets reappear; as a side effect the ground rain splashes also come back. |
| t9 | Made the discard conditional (`raining` + `open sky` + bluish pixel) so rain splashes are still hidden while indoor tap droplets survive. Documented edge case: an outdoor tap during rain still loses its droplets. |
| t12 | Released here: particles drawn through the translucent pipeline with in-shader occlusion, premultiplied alpha and matching blend functions (1.21+ translucent/particle fix), particle material masks so glowing particles emit, parameterised directional lightmaps, separate-entity-draw support gated for Iris 26.1+ with a dithered fallback, and the `PUDDLE_MODE` / `TRANSLUCENT_ALPHA` / `PARTICLE_OCCLUSION` options. |
