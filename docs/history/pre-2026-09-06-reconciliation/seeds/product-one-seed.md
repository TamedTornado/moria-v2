# Product One Seed — "The Walkable World"

*Scope: the voxel substrate, one generated region, and a character who can run around in it. No System, no spells, no gas, no game. This is the tech proven as a product-shaped demo. Everything here references voxel-world-substrate.md; this doc pins down what gets built first, what the seed world contains, and what "done" looks like.*

---

## 1. Product Statement

A generated natural world — hills, forest, river, cliffs, caves — that you can run through in third person, where everything you see is voxel truth: you can walk to any point, the terrain is continuous and smooth, and a debug key proves it's all mutable matter underneath. The demo's job is to make one claim undeniable: **this is not a heightmap with props — it's a fully material world, and it looks good.**

### Non-goals (explicitly out)
- Combat, stats, entities beyond the player, AI of any kind
- The System / LLM anything
- Gas, pricing, intent
- Building UI, blueprints, mechanisms
- Fluids beyond static bodies (tier 1 only — lakes and a river *channel* with a water surface; no flow sim)
- Weather sim, seasons, growth (a fixed time-of-day slider is enough)
- Persistence beyond "reload the same seed + deltas" (single save slot, no versioning)

### The one indulgence kept in
**Dig and place stay in scope**, on a debug key. Not as gameplay — as the *proof*. The demo clip is: run through a postcard forest, stop, carve a tunnel into the hillside mid-sprint, walk through it, and the cut faces look like cut earth. Without this, the demo is indistinguishable from any Unity terrain scene. With it, it's a substrate.

---

## 2. The Seed World

One authored *seed*, not one authored *world* — the region is generated, but the generation parameters are curated so the demo route reliably contains every proof point.

### Region
- **1km × 1km × 256m** (surface at ~64m, so ~64m of sky headroom in play, ~190m of geology below). Small enough to tune, large enough that sparsity and streaming are real (the whole region must NOT fit in memory as raw voxels — that's a test, not a limitation).
- **25cm voxels, 16³ bricks** per the substrate doc. Voxel-size final call happens here — this region is the benchmark bed for the 25cm vs 12.5cm question.

### Terrain composition (what the generator must produce from this seed)
| Feature | Why it's in the seed |
|---|---|
| Rolling meadow with grass dressing | The "reads as a normal world" claim, and the instancing pipeline proof |
| Dense mixed forest (2 tree species + bushes) | Voxel-object layer at count; canopy density stress |
| River with carved channel + a lake | Tier-1 fluid bodies; the carve pipeline proving rivers occupy real channels |
| Cliff face / rocky outcrop with exposed strata | Dual contouring sharp features; tilted strata visibly reading as geology |
| One karst cave system, walkable from a surface mouth to ~-40m | Deep-Z is real; underground rendering + light; homogeneous-brick sparsity at depth |
| One aquifer band and one ore vein type crossing the cave | Dig-down honesty — hit something *true* underground |
| Boulders, stumps, scattered rocks | Micro voxel objects |
| A small ruin (hand-stamped blueprint, placed by POI metadata) | The stamp/prefab path exercised once; sharp masonry vs organic terrain in one frame |

### Materials palette (seed set, ~14)
air, water, topsoil, subsoil, sand, gravel, limestone, sandstone, shale, granite, iron-ore, wood, leaf, cut-stone (ruin). Each with hardness, granular flag (sand/gravel), and render/splat properties. No burn/wet state rules yet — the state nibble exists in the format but no CA consumes it in product one.

---

## 3. Substrate Slice Included

From the layering diagram, product one builds the bottom two layers plus a sliver of the third:

**Generation layer — full.** Continent pass can be stubbed to "this one region's curated parameters," but columns, strata, caves, ore, lazy materialization, and POI metadata all ship as designed. This layer is the reusable asset; don't cheapen it.

**Matter layer — partial.**
- Brick pool, homogeneous sentinels, lazy materialization: full.
- Meshing (surface nets/dual contouring, GPU, dirty-brick incremental): full — this is the headline tech.
- Grass/clutter dressing from scatter points: full.
- Voxel objects (trees, boulders): placement, registration, rendering — **yes**; felling/rigid conversion — **no** (stretch goal, it's the best clip but it drags in the physics coupling).
- CA, fire, fluids-tier-2/3, integrity, granular settle: **no.** Format supports them; nothing runs them.
- Static water surfaces (tier 1): yes.

**Script/API layer — sliver.** The dig/place verbs and mirror queries exist as engine-internal API (the debug tools call them), establishing the "nothing touches voxels directly" boundary from day one. No embedded scripting language yet.

---

## 4. The Player

- **Third-person character controller**: run, sprint, jump, swim (surface paddle), collision against the *voxel truth* (capsule vs. brick occupancy, not vs. render mesh — this matters: it proves the mesh is a view).
- **Camera**: free-orbit third person for the demo — the Diablo lock comes with the ARPG, and a free camera sells the world better. Underground: camera collision + a simple light attached to the character.
- **Traversal must exercise Z**: walkable cave route, climbable-by-jumping rock shelves, the ruin's intact staircase. If the player can go from canopy-level cliff top to -40m cave floor in one continuous run, the continuous-3D claim is made.
- **Debug palette** (keys, not UI): dig sphere, place material sphere, wireframe/brick-boundary view, raw-voxel view toggle, streaming-ring visualizer, time-of-day slider.

---

## 5. Performance Targets (the actual product spec)

This product's customers are future-you and the audience on X; both buy numbers.

| Metric | Target |
|---|---|
| Frame rate | 60fps at 1440p on a mid GPU (3060-class); 60fps at 1080p–1440p on the M4 Mac Mini dev machine (bandwidth-bound — if the M4 hits this, discrete targets are nearly guaranteed) |
| Dig-to-remesh latency | dirtied bricks remeshed within 2 frames; no hitch on a 3m-radius carve |
| Cold-start into world | < 5s to walkable (lazy materialization doing its job) |
| Memory | full region under ~2GB GPU resident with streaming rings active; idle wilderness near-zero per the sentinel design |
| Save/load | delta save < 50MB after heavy defacement; load restores exactly |

Benchmarks are part of the deliverable: a scripted flythrough + carve-storm scene that outputs these numbers **plus a machine profile**, so every subsequent substrate change regression-tests against product one and numbers from different hardware stay comparable.

### Dev-platform constraints (M4 Mac Mini, 32GB unified, wgpu/Metal)
- **No 64-bit buffer atomics** — Apple GPUs don't support them. All counters, allocators, and label-propagation stay 32-bit (the design already complies; this pins it as a rule for future kernels).
- **Bandwidth is the ceiling, not compute** (~120–273GB/s depending on M4 variant). CA and meshing passes are traffic-bound: brick sparsity and homogeneous sentinels are load-bearing from milestone 1, not deferred optimization. Profile memory traffic first (Xcode Metal GPU capture works on wgpu apps and is excellent for this).
- **Unified memory softens the mirror problem** during dev — keep the FleX command/mirror architecture anyway; it's still correct for discrete GPUs and it's the sandbox/multiplayer boundary.
- **Stay on wgpu/WGSL** — no native Metal fork, ever, in the load-bearing layers. Portability to Vulkan/DX12 is the point of the crate.
- Discrete-GPU targets are unverifiable until the Linux box returns; treat them as provisional, re-baseline later.

---

## 6. Milestones (demoable, in order)

1. **Hill that looks like a hill** — meshing + triplanar over stub gen. *(First screenshot.)*
2. **Carve a smooth tunnel** — dig/place verbs, incremental remesh. *(First clip: the mid-sprint hillside tunnel.)*
3. **True geology** — strata, cave, ore, aquifer band visible in a cutaway debug view. *(The DF-audience clip.)*
4. **Dressed world** — grass, trees, boulders, river, ruin; the postcard. *(The trailer shot.)*
5. **The run** — controller + camera + the continuous cliff-top-to-cave-floor route. *(The playable demo.)*
6. **Numbers** — streaming, persistence, benchmark scene. *(The credibility post.)*
7. *(Stretch)* **Timber** — one felled tree with rigid-body fall. *(The viral clip, if the physics coupling comes cheap.)*

With the Visionary pipeline and parallel agents on well-partitioned work (gen layer, meshing, controller, and dressing are nearly independent), this is a **2–3 week build to milestone 6**, with the honest risk concentrated in milestone 2's remesh latency and milestone 6's memory numbers — everything else is assembly of known techniques.

## 7. What Product One Buys

- The substrate crates exist, benchmarked, with the API boundary enforced from the first commit.
- A public artifact for the X audience thread — each milestone is a post; milestone 5 is a downloadable demo.
- The decision bed for every open question in the substrate doc (voxel size, LOD strategy, object-layer scaling) answered with measurements instead of guesses.
- Product two (pick your poison: fortress-mode toybox, or the ARPG with the System) starts from a walkable world instead of a whiteboard.
