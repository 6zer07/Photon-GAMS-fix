<br><br>

<h1 align = "center">Photon GAMS — fixed (t12)</h1>

<p align = "center">A gameplay-focused shader pack for Minecraft, community-fixed fork of
<a href="https://github.com/OUdefie17/Photon-GAMS">OUdefie17/Photon-GAMS</a></p>

![Screenshot](docs/images/a.png)

## ⚠️ About this fork

This repository is a **community-fixed fork**. It continues
[OUdefie17/Photon-GAMS](https://github.com/OUdefie17/Photon-GAMS) (whose last upstream push was
2025-03-30), which in turn is based on [sw-52/gluon](https://github.com/sw-52/gluon) and
[sixthsurge/photon](https://github.com/sixthsurge/photon).

This fork's `main` branch carries the **t12** fix set: 20 shader files changed, all of them listed
below with their reason. Everything outside `README.md`, `README.zh_CN.md`, `CHANGELOG.md` and
`docs/t12-code-changes.patch` is **byte-identical to the tested `t12` build** — no other file was
touched.

* Full per-file history: [CHANGELOG.md](CHANGELOG.md)
* Raw unified diff against upstream `main`: [docs/t12-code-changes.patch](docs/t12-code-changes.patch)
* 中文说明: [README.zh_CN.md](README.zh_CN.md)

## What t12 fixes

### 1. Transparent textures and particles on 1.21+ (premultiplied alpha)

Translucent programs used to write straight (non-premultiplied) colour while the pipeline blends
with `ONE / ONE_MINUS_SRC_ALPHA`, so on versions that use premultiplied translucency those layers
came out too dark, too bright or effectively invisible. `gbuffers_textured`, `gbuffers_particles`
and the particle/entity/block translucent programs now output **premultiplied alpha**
(`PREMULTIPLIED_ALPHA`), the legacy `a = sqrt(a)` + `rgb / max(a, eps)` pair that fought the new
blend mode is gone, and the blend functions were updated to `ONE / ONE_MINUS_SRC_ALPHA`. This
matches Photon 1.3 behaviour. `gbuffers_weather` premultiplies rain and snow the same way.

### 2. Particle depth and particle glow

* Particles are now drawn through a program that writes to the **translucent layer** (`colortex13`)
  instead of the deferred gbuffer (`world*/gbuffers_particles.{vsh,fsh}`), so their blending and
  glow survive. As opaque deferred geometry they lost their blending entirely — which is what broke
  modded particles such as Ars Nouveau's ritual effects.
* That layer is composited **without a depth test**, so particles hidden behind solid geometry have
  to be rejected inside the shader. New option **`PARTICLE_OCCLUSION`** (default `ON`) discards them
  with `if (depth1 < gl_FragCoord.z - 1e-5) discard;`.
* The held item/tool no longer lets particles shine through it: `gbuffers_hand` writes
  `colortex13 = 0` wherever it passes the depth test, erasing the translucent layer behind the
  hand, and water behind the held item is fixed by the same change.
* Particles now get their material mask in the translucent vertex stage — `27` (particle) and
  `47` (glowing particle) — so **glowing particles actually emit** instead of being shaded as an
  unclassified material.
* `get_directional_lightmaps()` now takes the scene position as a parameter instead of reading the
  global `scene_pos`. In the translucent/particle programs that global holds the wrong value, which
  produced wrong screen-space derivatives and therefore wrong directional lightmaps (the reported
  glow/lighting errors).

### 3. Mod compatibility: bluish particles are no longer blanket-deleted

Upstream hardcoded

```glsl
// Kill the little rain splash particles
if (base_color.r < 0.29 && base_color.g < 0.45 && base_color.b > 0.75) discard;
```

which removed **every** bluish particle in the game — including Kaleidoscope Tavern tap/dishes
particles and animated block models, so those animations played with missing particles. The filter
now only runs while it is actually raining under open sky:

```glsl
if (rainStrength > 0.05 && light_levels.y > 0.1
    && base_color.r < 0.29 && base_color.g < 0.45 && base_color.b > 0.75) discard;
```

Rain splashes on the ground are still removed; indoor and bluish modded particles survive.

### 4. New settings

| Setting | Default | What it does |
| --- | --- | --- |
| `USE_SEPARATE_ENTITY_DRAWS` | off | Exposes Iris' `separateEntityDraws`. Needs **Iris 26.1+** — enabling those programs on older versions renders translucent entities and blocks incorrectly. |
| `DITHERED_TRANSLUCENCY_FALLBACK` | on | Dithered transparency for objects that should be translucent while separate entity draws are unavailable or off. |
| `PARTICLE_OCCLUSION` | on | Discard particles hidden behind solid geometry. Turn **off** to keep effects that mods intentionally draw without depth testing. |
| `PUDDLE_MODE` | full coverage | `patchy` = original random puddles, `full coverage` = the whole wet, flat, outdoor ground. |
| `TRANSLUCENT_ALPHA` | 0.75 | Opacity multiplier for translucent surfaces (modded wings, glass-like quads…), never applied to lightning (material 102) or nether portals (62). |

### 5. Other changes

* Puddles: `f0` 0.02 → 0.2, roughness reworked, **leaves excluded** (material 5), flat-normal test
  tightened, and the "indoors" test changed from `pow5(skylight)` to a linear step at 14/15 so
  puddles stop appearing under overhangs.
* `alphaTestRef` is read from the uniform instead of using a hardcoded `0.1` in the alpha tests.
* `IS_IRIS` compile-time branching (damage overlay, armor glint, deferred clear) is replaced by
  `USE_SEPARATE_ENTITY_DRAWS`, so it describes the feature actually in use rather than the loader.
* `particles.ordering = mixed` is pinned in `shaders.properties`, and the `alphaTest.*` block is
  commented out (defaults) instead of forcing `off` for every program.

## Known limitations

* The rain-splash filter works from colour and environment only — the shader cannot see particle
  types. A tap drip that runs **outdoors while it is raining** is still removed.
* `PARTICLE_OCCLUSION = ON` hides the "shines through everything" look that some mods rely on
  (Ars Nouveau ritual helix effects). Set it to `OFF` for those.
* Disabling translucent entity draws can make translucent objects opaque (white banners, food on
  campfires not rendering) — that is the trade-off documented by the option itself.

## Installation

1. Download this repository (Code → Download ZIP) or clone it.
2. Put the folder — or a zip of its contents — into `.minecraft/shaderpacks/`.
3. Select it in **Video Settings → Shader Packs** (Iris). Reload with `R` after changing options.

The premultiplied-alpha path in this fork is what makes translucent textures and particles work on
1.21+; on 1.20.1 it matches the upstream look.

## Acknowledgments

* OUdefie17 (Mod support, Colored Light settings, Some features)
* Arona74  (Mod support, Transfer to new versions of Photon, Owner of the dev branch)
* -Daytendo64- (Galaxy, Nebula, Shooting Stars, End Solar Flare settings)
* sw-52 (More Tonemap Operators and settings, Fog settings, DOF settings)
* The t12 fix set was developed and tested by the maintainer of this fork.

### Features

* Better Hardcoded Emission
* More customization options
* More settings for Colored Light, Water, Sky, Fog
* Better supprort for mods
* More Tonemap Operators

## Community

For questions, suggestions and news regarding this shader pack, head to
[Photon discord server thread](https://discord.com/channels/1007736612488220724/1288402151097499698)

## License

Photon GAMS follows Photon by Sixthsurge. The original [LICENSE](LICENSE) is kept unchanged in this
repository and applies to the whole pack, including this fork's modifications: you may modify and
redistribute it, but you may **not** publish it or derivative works on mod-sharing platforms that
provide financial benefits to creators (Modrinth, CurseForge, …) without the original author's
written permission, and you may not sell it.
