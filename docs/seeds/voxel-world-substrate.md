# Voxel World Substrate — Design Specification

*Companion to
[`gpu-resident-substrate.md`](gpu-resident-substrate.md). This is a broad
architecture reference for natural-world consumers of the voxel substrate. It
contains game examples and future extension ideas that are nonbinding unless
selected by [`project-boundary.md`](project-boundary.md) or a later explicit
decision. Moria itself has zero game or LLM dependency.*

---

## 1. Design Goals

1. **Reads as a normal world.** Rolling terrain, forests, rivers, cliffs, meadows. Not a Minecraft aesthetic — the voxel grid is the *truth*, not the *look*.
2. **Mutable everywhere, all the way down.** Any voxel can be destroyed, moved, or placed. Dig a mine under your house. Collapse a cliff into a river. Nothing is decorative geometry sitting outside the material world.
3. **Deep Z is first-class.** The underground is content, not a skybox floor. Cave systems, strata, ore, buried ruins, and the Moria fantasy: descending levels of increasingly dangerous depth.
4. **Substrate, not game.** Clean layering so the same crate stack supports: the System ARPG, a DF-style fortress/colony game, a Moria-style descent roguelike, or a pure sandbox. Game rules live above the substrate; the substrate provides matter, physics, queries, and mutation.
5. **GPU-resident architecture direction** per
   [`gpu-resident-substrate.md`](gpu-resident-substrate.md): sparse brick
   storage and a command/query boundary that can support asynchronous GPU work.
   Specific simulations remain nonbinding until selected for a Moria milestone.

---

## 2. The Look Problem: Voxels That Don't Look Like Voxels

"Normal-looking surface world" and "voxel truth" are in tension only if voxels are rendered as cubes. Three viable strategies:

### Option A — Small voxels, rendered raw (Teardown look)
~10–25cm voxels raymarched directly. Chunky-but-charming. Cheap to implement, expensive in memory at overworld scale, and the aesthetic is unmistakably "voxel game." Rejected as primary for the surface — acceptable for debug view.

### Option B — Smooth isosurface extraction (recommended for terrain)
Voxels store **material + density** (or signed distance in a narrow band near surfaces). A meshing pass — **surface nets or dual contouring** over each dirty brick — extracts a smooth triangle mesh. Dual contouring preserves sharp features (cliff edges, cut faces, building corners) that marching cubes rounds off, which matters because *player cuts should look like cuts*.

- Meshing runs as a **GPU compute pass over dirty bricks only**, output into a per-brick mesh pool. A 16³ brick meshes in microseconds; a fireball dirtying 30 bricks re-meshes within a frame or two.
- Materials blend at boundaries via triplanar texturing + material-ID splatting from the voxel data. Grass-to-dirt-to-rock transitions look painterly, not tiled.
- The mesh is a *view*. All physics, queries, gas metering, and gameplay run against voxel truth. The mesh is regenerated, never authoritative, never saved.

### Option C — Hybrid (the actual answer)
- **Terrain and structures**: Option B smooth extraction.
- **Player/System-built constructions**: dual contouring already preserves their sharp edges; optionally a per-material "crystalline" flag forces hard-face extraction so masonry looks like masonry while dirt looks like dirt.
- **Vegetation and clutter**: *not* extracted from terrain voxels at all — see §5. They are voxel-backed objects with their own representation.

Voxel size for the world grid: **25cm** (4 voxels/meter). Fine enough that craters, tunnels, and carved stairs look organic after smoothing; coarse enough that a 2km × 2km × 512m region is tractable under sparsity (the overwhelming majority of bricks are "all air" or "all stone" and stored as single palette values — see §3).

---

## 3. Storage: Bricks, Palettes, Columns

Extends the two-level brick-grid direction in
[`gpu-resident-substrate.md`](gpu-resident-substrate.md) with what a *natural
world* specifically needs.

### 3.1 Brick pool
- **16³ voxel bricks** (4m cubes at 25cm). Top-level table maps brick coordinates → pool index or a **homogeneous sentinel** (this entire brick is air / stone / water — one palette entry, zero pool cost).
- Homogeneous bricks are the workhorse: untouched underground is solid-stone sentinels; sky is air sentinels. Only the *interesting shell* — surface, caves, structures, player scars — occupies real pool memory. This is the VDB insight doing its job.

### 3.2 Per-voxel payload (8–16 bits, packed)
| Field | Bits | Purpose |
|---|---|---|
| material ID | 8 | index into material palette (256 materials is plenty per world; palette is per-world, System-extendable) |
| density/occupancy | 4 | isosurface weight for smooth meshing; 0 = empty, 15 = full. Digging *erodes* density before deleting — half-dug voxels look half-dug |
| state nibble | 4 | material-defined: wetness, burn stage, growth stage, damage |

Optional second channel (sparse, only allocated for bricks that need it): temperature, contaminant/fluid ID, light level for underground.

### 3.3 Per-brick aggregates (the coarse sim grid, extended)
Material histogram, mean temperature, wetness, integrity, **flammable fraction**, **diggable-hardness class**, fluid volume, vegetation density, and a **support flag set** for structural integrity (§8). These aggregates *are* the coarse mirror the CPU/scripts read, and *are* the CA cells for propagation. One structure, three consumers.

### 3.4 Column index (the DF concession)
Alongside the 3D brick table, maintain a lightweight **2D column index**: per (x,y) column, a run-length list of material bands (air 0–62m, soil 62–65m, stone 65m–bottom, plus cave gaps). Purposes:
- O(1) "what's the surface height here" for spawning, vegetation, pathfinding seeds, rain.
- DF/Moria mode: instant Z-slice views ("show me level -12") without touching bricks.
- Worldgen writes columns first, bricks materialize lazily from column data on first access (§4.5).

The column index is derived data — rebuilt from bricks when dirty — but it's the reason DF-style games sit *comfortably* on this substrate instead of fighting it.

---

## 4. World Generation: Geology First

A world that digs well must be generated as **geology**, not as a heightmap with rock painted underneath. Layered pipeline, each stage a pure function over coordinates + world seed, so any brick can be generated independently and lazily:

### 4.1 Continent & climate pass (coarse, 1 sample / 64m)
Tectonic-ish plates via jittered Voronoi → base elevation, mountain arcs, ocean basins. Latitude + elevation + rain-shadow → temperature/moisture fields → **biome map** (Whittaker-style lookup). This pass is small enough to precompute for the whole region and hand to the System as a *map it can read and annotate*.

### 4.2 Terrain pass (per column)
Elevation = continent base + ridged multifractal (mountains) + billow (hills) + domain-warped fine noise. Rivers: descend-the-gradient carving from spring points to sea level, with valley widening — rivers must *actually occupy carved channels* because players will drain, dam, and divert them (§7).

### 4.3 Strata pass (per column, the DF heart)
Deposit **geological layers**: topsoil → subsoil → sedimentary bands (sandstone, limestone, shale, coal seams) → metamorphic → igneous basement, with layer thickness varying by noise and tectonic uplift *tilting* strata so mountainsides expose old layers. Ore veins as 3D ridged-noise threads confined to appropriate host rock (iron in sedimentary bands, gold in quartz veins in igneous, etc.). Every material carries **hardness** (dig cost / gas weight), so descent naturally gates on tools/power — the Moria progression is emergent from geology.

### 4.4 Cave & void pass (3D)
- **Cave systems**: 3D gyroid/worley noise thresholded within karst-capable strata (limestone) → natural cavern networks, plus worm-tunnel carvers for connectivity.
- **Aquifers**: water-saturated strata bands (DF players will feel seen and afraid).
- **Deep voids**: large caverns at depth, magma chambers near the basement — the substrate's "hell layer" hook. These are marked as **POI-capable volumes** in metadata the System can claim (buried ruin here, breeding warren there) without touching the geology code.

### 4.5 Lazy materialization
Nothing above writes bricks eagerly. A brick materializes on first *touch* (render proximity, sim activation, mutation, or query) by evaluating the pipeline for its 16³ cells. Untouched world costs only the column index and the coarse continent maps. Combined with homogeneous sentinels, an effectively boundless region idles at trivial memory.

### 4.6 Where the System attaches
The LLM never generates geology — noise is better and cheaper at rock. The System operates on the metadata layer: it *reads* the biome/strata/POI maps and *directs* — "place a fire-cult ruin in this basalt cavern," "this forest is corrupted, swap its vegetation palette and add a spread rule," "vein this region with the exotic material I just defined." It authors **placement, palettes, structures (as prefab voxel stamps or generator scripts), and new materials** — the same package model as spells: a material definition is data + optional CA rule script + optional kernel.

---

## 5. Vegetation & Surface Dressing

The rule: **everything that can burn, break, or block is voxel-backed. Everything else is dressing anchored to voxels.**

### 5.1 Trees — voxel objects
Trees are *not* terrain voxels and *not* decorations. Each tree is a **voxel object**: its own small brick set (trunk = wood material with density; canopy = leaf material, low density, high flammability) registered in the world grid via an object layer (grid cells reference object ID + local offset). Why objects rather than baked-into-terrain:
- **Future consumer concept, not current Moria scope — falling.** A downstream
  game could convert a disconnected tree into a rigid-body proxy through a
  separately specified particle/rigid layer. Moria does not acquire tree
  felling, rigid conversion, damage, or re-voxelization requirements from this
  example.
- **Growth.** A tree object carries a growth script/stage; the state nibble drives canopy expansion by swapping stamped voxel patterns. Saplings → trees over game time; the System can author new species as (voxel stamp generator + growth rule + material entries).
- Generation: species chosen by biome, placed by Poisson-disk on the column index, stamped lazily with the brick they'd occupy.

### 5.2 Bushes, boulders, stumps — micro voxel objects
Same object mechanism, smaller. A boulder is a granite voxel blob that can be split (CSG), rolled (rigid proxy), or absorbed into a build. A bush is 2–3 flammable leaf bricks. Everything the player would expect to interact with, is interactable, because it's all the same matter system.

### 5.3 Grass, flowers, ground clutter — instanced dressing
Pure GPU instancing driven by *voxel data*: the meshing pass emits "scatter points" on upward-facing surface triangles whose material is grass/dirt-with-cover, modulated by the brick's vegetation-density aggregate. Grass has **no individual voxel identity** but is *not* fake:
- Fire in a brick → brick's burn state rises → scatter pass swaps instances to charred/none. Grass burns because the *ground* burns.
- Digging removes the surface voxels → scatter points vanish with the surface they lived on.
- Trampling/mowing: writes the state nibble on surface voxels; dressing responds.
Cost: near-zero memory, one indirect draw, and it never desyncs from the matter world because it's a pure function of it.

### 5.4 Snow, sand, gravel — granular materials
Materials flagged **granular** get a cheap CA settle rule (unsupported granular voxels slump to neighbors / convert to falling particles). Sand pours, gravel collapses in mines, snow accumulates as a thin density layer written by the weather system onto exposed surface voxels. This is the 20%-of-DF-terror (cave-ins from digging into sand) that makes underground play honest.

---

## 6. Water & Fluids

Full per-voxel fluid CA at 25cm over kilometers is not happening, and doesn't need to. Three-tier scheme:

1. **Bodies** (lakes, seas, still reservoirs): stored as *volumes* — per-column water level in the column index + brick-level water material fills. Zero sim cost while undisturbed. Rendering is a surface at the level plane.
2. **Coarse flow** (rivers, floods, drained dams): CA on the **brick aggregate layer** — per-brick fluid volume + flow momentum between the 6 neighbors, DF-style pressure rules. Runs only on *active* fluid bricks (sparsity again). When the player breaches a dam, the affected bricks activate, the flood propagates at brick resolution (4m cells — coarse but dramatic and correct), and fine voxel water levels within each brick are interpolated for rendering/interaction.
3. **Future consumer concept, not current Moria scope — fine splash.** A
   separately specified dynamic-matter layer could emit particles at fluid
   boundaries and settle them back into brick volumes.

Interactions route through material rules: water brick-flow into a fire brick quenches it (state nibble), into a magma brick creates obsidian voxels + steam particles (the DF classic, essentially free once the rule table exists), through a soil brick raises wetness → mud material swap. **Aquifer breach** = digging opens a face into a saturated stratum → that brick joins the active fluid set with inflow. The Moria/DF building game gets its full hydrological toybox — wells, channels, floodgates, magma forges — from these three tiers plus player-placeable gate/pump mechanisms (§9).

---

## 7. Weather, Time & Ambient Simulation

Thin but present, because the surface world reads as "normal" only if it *behaves*:
- **Day/night + seasons** driving light, growth-rule ticks, and snow-line elevation.
- **Weather fronts** as region-level states (the System can also *author* weather as events): rain writes wetness to exposed surface bricks + fills water tables; storms add lightning strikes (ignition events); drought lowers body levels.
- **Fire ecology**: lightning + dry-season wetness floor → natural wildfires that the brick CA propagates and rain extinguishes. The world demonstrates the matter rules *to* the player before the player exploits them — Noita's legibility principle at landscape scale.
- A downstream game may use aggregates or columns outside active voxel range.
  Any System or LLM adjudication is a game-layer concern and is not part of
  Moria.

---

## 8. Structural Integrity & Cave-ins

The following is a future building-game extension, not a current Moria
requirement:

- **Support graph at brick granularity**, refined per-voxel only in boundary bricks. Each solid brick tracks support class: *grounded* (connected to basement via solid bricks), *anchored* (connected to grounded within material-dependent span), *unsupported*.
- **Material span tables**: wood beams span further than stone, stone further than dirt, granular materials span ~0. This single table is the difference between "physics puzzle" and "building game" — players learn that oak beams let them roof a 6m hall and iron lets them do 12m.
- **Failure**: unsupported region → convert to falling rigid/debris (dust particles, damage on impact, re-voxelize as rubble material). Cascades naturally: knock out a pillar, the roof section falls, its load-path neighbors re-evaluate next tick, maybe more falls. Amortized, dirty-region, GPU label-propagation.
- **Deliberate engineering**: because support is queryable through the mirror, the DF-style game can show players a support overlay, and the System can *write monsters that read it* — the siege beast that targets your load-bearing pillar is just a script querying the same aggregate every player tool queries. Symmetry of substrate, again.

Tunables per game: the ARPG wants forgiving spans and dramatic collapses; the fortress game wants honest engineering. Same system, different table.

---

## 9. Building: Placement as First-Class Verb

Digging's mirror twin, and the half that unlocks DF/Minecraft modes:

- **Voxel placement API**: place material into empty voxels, gas/resource-priced by material. Snapping modes: free-form (organic, sculpting), grid-aligned (masonry), and **stamp** (prefab multi-brick shapes: wall segments, arches, stairs, door frames).
- **Blueprints**: a blueprint is a sparse voxel stamp + material manifest. Sources: player-designed (in-game copy tool over a region — "copy my gatehouse"), System-authored (the System designs a structure as a stamp — this is *exactly* how it places ruins in worldgen, so blueprints and world structures are one format), or shared between players. In fortress mode, blueprints + a manifest become **work orders** for agent labor.
- **Mechanisms**: doors, hatches, floodgates, levers, pumps as *entity objects occupying voxel footprints* — they participate in the support graph and fluid boundaries but carry entity logic (scripts). A floodgate is a voxel-shaped entity whose script toggles its footprint between solid and open. This is the entire DF machinery layer expressed in the existing object + script model.
- **Structures as semantic regions**: a flood-fill room detector over enclosed volumes (walls + roof + door) tags **rooms** in metadata — required for fortress gameplay (assign bedroom, designate workshop), for the ARPG's town-value economy (a "town" is a set of intact rooms with residents — the thing your collateral damage destroys has a *ledger*), and for the System's spatial reasoning ("host has constructed a fortified structure at…").

---

## 10. Entities, Pathfinding & the Z-Axis

- **Nav data derived from bricks**: per-brick walkability aggregate (has standable surface, headroom, hazard flags) forms a coarse nav graph; fine within-brick paths computed on demand. Dirty bricks invalidate their nav node — mutation-safe pathfinding without global rebuilds.
- **3D movement classes**: walkers (need floor + headroom), climbers (traversable vertical faces by material), fliers (air volumes), burrowers (path *through* diggable material below a hardness threshold — the thing that makes underground sieges terrifying and is nearly free given hardness is already per-voxel), swimmers (fluid volumes).
- **Z-levels as gameplay**: the column index + nav graph make DF's "levels" a *view and designation convenience*, not a world structure — the world is continuous 3D, but fortress mode can present and designate in slices.
- Agent labor (fortress mode): work orders (dig designation, build blueprint, haul) are queries + mutations through the same API spells use. **Dwarves are gas-free scripts.** The substrate does not distinguish a miner from a magic missile — both are agents mutating matter through priced verbs; only the pricing policy differs per game.

---

## 11. Persistence & Streaming

- **Truth = worldgen function + edit deltas.** A brick saves nothing if untouched; touched bricks save a compressed delta (palette-compressed voxel diff) keyed by brick coordinate. Scars are cheap: even a heavily-mined fortress is thousands of bricks, megabytes.
- **Object & entity journals**: trees felled, objects moved, entities and their script state — event-sourced per region.
- **Streaming**: rings around active cameras/agents — *render ring* (meshed + dressed), *sim ring* (CA + fluids + integrity active), *aggregate ring* (mirror only, ambient sim on aggregates), *cold* (column index + deltas on disk, System-adjudicated). Fortress mode keeps the sim ring pinned around the fortress even when the adventurer half of a Moria-style game wanders off — rings are per-anchor, not per-camera.
- **Cross-run persistence** (ARPG hub model or DF fortress→adventurer reuse): deltas are the save. Your abandoned fortress *is* a delta set the next mode loads as a dungeon. This substrate makes DF's legendary "reclaim your dead fortress" loop a file-format feature.

---

## 12. Layering: The Substrate as Reusable Crates

```
┌─ game layer ──────────────────────────────────────────────┐
│  ARPG (System, spells, gas policy)  │  Fortress  │  Moria │
├─ semantic layer ──────────────────────────────────────────┤
│  rooms/structures, nav, work orders, blueprints,          │
│  economy hooks, designation views (Z-slices)              │
├─ script/API layer ────────────────────────────────────────┤
│  priced verb set, mirror queries, events, object model,   │
│  mechanism entities, gas policy plug-in                   │
├─ matter layer ────────────────────────────────────────────┤
│  brick pool, CA rules, fluids, integrity, granular,       │
│  fire ecology, particle coupling, meshing/dressing        │
├─ generation layer ────────────────────────────────────────┤
│  geology pipeline, biomes, columns, lazy materialization, │
│  POI metadata, material palette registry                  │
└───────────────────────────────────────────────────────────┘
```

Rules of the layering:
- **Gas is a policy object**, injected at the script/API layer. The ARPG prices verbs in mana-gas; fortress mode prices them in labor-time and materials; sandbox mode prices them at zero. Same verbs, same substrate.
- **The System is a game-layer client**, not a substrate feature. It consumes the same mirror, emits the same commands, authors content through the same registries (materials, blueprints, scripts, kernels) that hand-authoring uses. Any game on the stack can add or omit it.
- **Nothing above the matter layer touches voxels directly.** Everything goes through verbs and queries — which is simultaneously the sandbox boundary, the multiplayer-readiness boundary, and the reason the substrate stays reusable.

---

## 13. Build Order (vertical slice for the substrate itself)

1. Brick pool + homogeneous sentinels + lazy materialization from a stub 2-layer worldgen (heightmap + stone). Debug raw-voxel view.
2. Surface-nets meshing pass + triplanar material rendering. *Milestone: a hill that looks like a hill.*
3. Dig + place verbs, density erosion, dirty-brick remesh. *Milestone: carve a smooth tunnel, build a crude wall.*
4. Geology pipeline (strata, caves, ore) + column index. *Milestone: dig down and hit limestone, a cave, an aquifer scare.*
5. Brick CA (fire, wetness) + grass dressing + one tree species as voxel object with felling. *Milestone: burn a meadow, chop a tree onto a goblin.*
6. Fluids tier 1+2. *Milestone: breach a pond into your mine.*
7. Integrity + granular. *Milestone: an honest cave-in.*
8. Persistence deltas + streaming rings.

Each milestone is independently demoable, which matters for the X audience-building thread as much as for validation — steps 3, 5, 6, and 7 are each a viral clip on their own.

## 14. Open Questions

- Voxel size final call: 25cm assumed here; 12.5cm doubles fidelity and octuples cost — possibly per-region (fine near settlements, coarse wilderness)?
- Chunked marching-cubes LOD for distant terrain vs. imposter cards from the column index — how far does the Diablo camera actually let us cheat?
- Object layer capacity: at what tree count does the object registry want its own spatial acceleration structure?
- Whether fluids tier 2 needs a pressure solve (DF-accurate U-bend behavior) or momentum-only is enough for both games.
- Multiplayer: the verb/command architecture is server-authoritative-ready by construction — worth keeping in scope statements even if not built.
