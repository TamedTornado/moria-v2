# Moria product vision

Standalone product vision for the Moria voxel-world substrate. A downstream
product designer can design consumer contracts, acceptance criteria, and the
public facade from this document alone—without reopening seed material or
deciding which source is authoritative.

**Authority.** The human-approved project scope boundary is authoritative for
what is current, adjacent, future, excluded, and unresolved. Every claim below
is within that boundary. Adjacent or future material may explain why a
capability exists; it is not a current deliverable. Excluded material does not
appear as product requirement.

---

## 1. Product identity

**Authority: C-001, C-002, C-012, C-014, AD-001, AD-004, AD-006.**

**Moria** is a reusable **voxel-world substrate**: engine-shaped world
infrastructure that downstream games and tools install and drive through a
public consumer contract.

It is:

- A **Rust and Bevy library** (or small family of tightly scoped crates) for
  crate consumers—not an ecosystem-neutral engine abstract and not a shipped
  game.
- Infrastructure for continuous three-dimensional **material volumes**—natural
  landscapes, underground geology, and constructed interiors among them—without
  limiting identity to a Minecraft-style cube aesthetic or a single overworld
  content palette.
- **Volume-general:** substrate contracts must not assume gravity-aligned
  planetary terrain as the only legal world shape. Freeform hulls (ships,
  stations, multi-deck interiors) remain **future-consumer motivation**, not
  current delivery or validation targets; contracts must stay able to express
  them later.

It is **not**:

- A playable game, game mode, progression loop, or game-rules stack.
- A baked-in procedural or deterministic world generator.
- A baked-in physics engine (hand-rolled or third-party).
- Defined by any single validation harness, demo route, character controller,
  content postcard, or machine-specific demo target.

---

## 2. Purpose

**Authority: C-003, C-008, C-010, C-012, C-016, AD-003, AD-007.**

Voxel worlds only work when several hard systems agree as **explicit, shareable
contracts**: sparse material truth, bounded inspection, mutation admission,
streaming lifecycle, collision against matter rather than presentation,
persistence of world matter plus edits, GPU-resident representation of that
matter, measurable presentation derived from truth, and generic seams so
external behavior can plug in without privileged access to voxel storage.

Moria exists so material worlds can be consumed without each game rebuilding
those contracts, and without hiding them behind a single privileged demo.

**The product’s infrastructural claim:** a consumer can obtain a continuous
three-dimensional material world, including movable material volumes, inspect
and mutate it only through supported interfaces, keep authoritative matter
**GPU-resident** for scale and gameplay-enabling work, and trust that what they
see and collide with is a view of the **same** authoritative matter. External
plug-ins may implement physics, damage, or other behavior by observing that
truth and requesting admitted changes, but the substrate does not prescribe
their concepts, state, rules, or outcomes.

**World generation is not part of the substrate identity.** How a game fills or
seeds sparse material volumes is a consumer- or game-dependent algorithm that
runs *on top of* the substrate. The substrate provides storage, query,
mutation, streaming, collision-truth seams, generic behavior-extension seams,
persistence seams, presentation derivation, and content-injection seams for
material truth—not a particular generator, physics engine, damage system, or
behavior vocabulary.

---

## 3. Who it serves

### Primary consumers

**Authority: C-001, C-013.**

- **Game and tool authors** who need continuous material volumes with honest
  mutation, inspection, collision truth, streaming, and persistence, without
  rebuilding world infrastructure for each title.
- **Adjacent in-repo harnesses** (curation, benchmark, visual validation,
  optional walkable proof) that exercise the same public interfaces an external
  game would use. They prove contracts; they do not redefine product identity
  or “done.”

### Needs the product must meet

**Authority: C-003, C-004, C-005, C-006, C-007, C-008, C-009, C-010, C-011, C-012, C-013, C-014, C-015, C-016, AD-002, AD-003, AD-005, AD-007.**

| Consumer need | What success means |
| --- | --- |
| One material world | Occupancy, queries, collision truth, persistence, and external behavior plug-ins all read the same authoritative matter—not a private mesh world or a second invented occupancy model. |
| Contracted access only | Install the facade; inspect through public reads; mutate through admitted edits; never require privileged internal paths for harnesses or external crates. |
| Scale without full residency | Large regions stay tractable: untouched homogeneous volume stays cheap; only the interesting shell and active edits pay detailed cost. |
| Everywhere mutation | Any material cell the contract exposes can be destroyed or placed; cut faces and scars remain honest matter; presentation rebuilds from truth. |
| Genuine volumetric depth | Volume along the full third axis is real content—caves, strata, ore, aquifers as material bands, buried structure—not a heightmap floor with painted underground. |
| Movable material volumes | Dynamic voxel volumes can move and be mutated under the same truth contracts as static geometry. External systems decide whether those changes represent physics, damage, or something else. |
| Behavior without ownership | External systems can observe authoritative matter and request admitted changes without privileged voxel access. They own their concepts, state, and rules. |
| Tractable scars | Material edits and related world-state scars persist without dumping every cell of an untouched volume. |
| GPU-scale work | Sparse representation and a command/query boundary keep world matter GPU-resident and support asynchronous GPU work without changing the consumer contract. |
| Measurable quality | Benchmarks and harnesses can evidence mutation response, streaming, GPU memory behavior, collision-truth honesty, and behavior-extension readiness when exercised—without redefining the product as a game. |
| Content ownership | Consumers inject or drive world content (including their own generation algorithms) through seams; no particular generator is substrate law. |
| Domain-appropriate look | Fully material volumes can present as coherent for the consumer’s domain (geology, masonry, interiors, etc.); no single overworld aesthetic is mandated. |

---

## 4. Product boundary

### This product owns

**Authority: C-003, C-004, C-005, C-006, C-007, C-008, C-009, C-010, C-011, C-012, C-013, C-015, C-016, AD-002, AD-003, AD-005, AD-007.**

The reusable substrate and its public facade:

1. **Sparse storage and lazy materialization** of voxel truth.
2. **GPU-resident sparse representation** and a **command/query boundary** that
   can support asynchronous GPU work without changing the consumer contract.
3. **Bounded world inspection** and **telemetry** for consumers.
4. **Mutation admission** (dig, place, and related world-edit verbs) so nothing
   touches voxels outside the contract.
5. **Streaming and lifecycle** so large regions do not require full raw-voxel
   residency.
6. **Collision and occupancy truth** against voxel matter, not against
   disposable meshes—queries and contacts read material authority so consumers
   and plug-ins do not invent a second world.
7. **Generic behavior-extension seams:** external systems can observe
   authoritative matter and lifecycle changes and request admitted mutations
   without privileged voxel access. Physics, damage, fracture, health, gravity,
   force, and similar semantics and state belong entirely to those external
   systems.
8. Support for **dynamic voxel volumes**—material volumes that can move and be
   mutated through the same truth contracts as static world geometry. What
   movement or mutation means is consumer behavior, not substrate policy.
9. **Persistence** of material truth plus edit deltas (and related world-state
   scars), without requiring a dump of every cell; how the *base* volume is
   produced is the consumer’s concern.
10. **Presentation support** that derives surfaces and surface dressing from
    material truth, without serializing derived geometry as authority.
11. **Object and clutter registration hooks** so vegetation, micro-objects, and
    other matter-backed assemblies can register without baking into a single
    terrain slab.
12. **Seams** a consumer can use to inject or drive world content (including
    generation algorithms), without embedding any particular generator as
    substrate law.
13. **Volume-general contracts** that do not encode planetary heightmap
    assumptions or gravity-aligned terrain as the only legal world shape.

### Adjacent (not product identity)

**Authority: C-008, C-012, C-013, AD-001, AD-006, AD-007.**

- **Curation, benchmark, and visual-validation executables** and similar
  harnesses. They may curate parameters, exercise contracts, capture evidence,
  and visually validate the substrate, but only through the same public
  interfaces available to an external game. Controllers, characters, cameras,
  authored demo routes, presentation polish, acceptance scenarios, and
  **game-specific generation algorithms** belong to those consumers—not to
  substrate identity.
- A **walkable-world harness** is a *permitted* adjacent artifact, not a
  mandatory definition of “done.” In-repo generation used only to exercise or
  demonstrate contracts remains a harness concern, not a claim that generation
  is substrate product.
- **Behavior systems such as physics and damage.** A hand-rolled or third-party
  system may integrate through the substrate’s generic extension seams, but
  Moria does not ship or own its behavioral vocabulary, state model, or rules.
  Gravity, force, contact response, health, resistance, damage types, fracture,
  crumbling, and similar behavior remain plug-in / consumer concerns. An
  optional proof may demonstrate that a plug-in can observe truth and submit
  admitted changes; it does not make any particular behavior model a substrate
  deliverable.

### Downstream (not this product)

**Authority: C-002, C-014, AD-004.**

Actual games and game layers, including but not limited to: player control,
characters, skeletal animation, game-specific presentation, combat rules, AI
behavior, economy, building policy, the System / LLM layer, spells, gas
pricing, agent labor, building UI / blueprints as gameplay, mechanisms as game
entities—and **which** procedural or authored generation pipeline and behavior
systems a game uses.

Freeform ship and station games (and their design/combat fiction) are
**future-consumer examples**, not current delivery or validation targets.
Compatibility seams may be designed where substrate requirements demand them;
those layers are not implemented as Moria product.

---

## 5. Product-level outcomes

These are outcomes the substrate must enable—not a feature inventory or
implementation plan.

### 5.1 Truth vs view

**Authority: C-003, C-011, C-016.**

Occupancy, queries, collision truth, and persistence run against voxel matter.
Meshes, surface dressing, and debug geometry are **derived and disposable**.
External behavior systems also consume material truth through public seams
rather than a private mesh world. Derived geometry is never
serialized as world authority.

### 5.2 Contracted consumption

**Authority: C-003, C-010, C-013.**

External consumers install the facade, inspect through public reads, mutate
through admitted edits, and never require privileged internal paths. Adjacent
harnesses and external game crates share the **same** public boundary. GPU work
may complete asynchronously; consumers must not depend on direct buffer access
or synchronous ownership of internal storage.

### 5.3 Sparse scale

**Authority: C-004.**

Large regions remain tractable: untouched homogeneous volume stays cheap; only
the interesting shell (surfaces, voids, structures, player scars) and active
edits pay detailed cost. Streaming and lifecycle keep residency bounded without
requiring full raw-voxel presence for an entire region.

### 5.4 Mutable everywhere

**Authority: C-005, AD-002.**

Any material cell the contract exposes can be destroyed or placed. Cut faces
and scars remain honest matter; presentation rebuilds from truth. Mutation is a
first-class product proof—not optional scenery decoration sitting outside the
material world.

### 5.5 Deep Z is first-class

**Authority: C-006, AD-002, AD-004.**

Volume along the full depth axis is real content—genuine volumetric depth, not
a heightmap floor with painted underground. Caves, strata, ore, aquifers as
material bands, and buried structure are material volume, not skybox scenery.
Contracts stay volume-general so non-planetary freeform volumes remain
expressible; ship and station interiors are future-consumer motivation, not a
required current deliverable shape.

### 5.6 Dynamic voxel volumes

**Authority: C-007, AD-005.**

The world is not static geometry alone. The substrate must support material
volumes that move and can be mutated, so games can represent moving matter
under the same truth contracts rather than as overlays disconnected from the
world. Physics, damage, and other rules that cause those changes remain
external behavior.

### 5.7 Behavior-extension seams, not behavior policy

**Authority: C-008, C-016, AD-007.**

External systems can observe authoritative matter and submit admitted changes
through stable public seams. The substrate does not define physics or damage
concepts, canonical behavioral fields, simulation state, or response rules. A
proof plug-in may demonstrate the seam without becoming a required engine or
behavior model. Collision and occupancy **truth** against voxel matter remains
a substrate concern so plug-ins and consumers share one material world.

### 5.8 Cheap scars over full dumps

**Authority: C-009, C-012.**

Persistence keeps material edits and related scars tractable—not a dump of
every cell. A consumer may choose a reproducible generation function as its
base-world strategy; that choice is game-dependent and is not a substrate
deliverable.

### 5.9 GPU-resident architecture

**Authority: C-010, AD-003.**

Sparse representation and a command/query boundary keep world matter
GPU-resident and support asynchronous GPU work. This is a product distinction
from CPU-driven voxel engines: residency enables gameplay-scale mutation,
meshing, and future simulation without abandoning the consumer contract.
Specific kernels and simulations remain design-selected later; residency and
the async-capable boundary do not.

### 5.10 Measurable substrate quality

**Authority: C-013, C-015.**

Benchmarks and harnesses can evidence mutation response, streaming, GPU memory
behavior, collision-truth honesty, behavior-extension readiness when exercised,
and related contracts without redefining the product as a game. Harness-side
generation or a proof behavior plug-in used for tests or demos does not make
generation, physics, or damage behavior a substrate requirement.

### 5.11 World-dependent presentation

**Authority: C-002, C-011, AD-002, AD-004.**

How a world “looks natural” depends on the consumer’s world—landscape geology,
fortress masonry, and other material styles. The substrate must support fully
material volumes that read as coherent for their domain; it does not mandate a
single overworld aesthetic or a heightmap-with-props look. Ship bulkheads and
similar freeform-hull presentation remain future-consumer context, not current
validation targets.

---

## 6. Capabilities (product altitude)

**Authority: C-003, C-004, C-005, C-006, C-007, C-008, C-009, C-010, C-011, C-012, C-013, C-014, C-015, C-016, AD-002, AD-003, AD-004, AD-005, AD-006, AD-007.**

What the product enables and guarantees, without choosing algorithms, layouts,
crate splits, or milestones.

| Capability | Guarantee |
| --- | --- |
| Sparse material truth | Continuous volumes are representable without full raw-voxel residency of untouched regions. |
| Lazy materialization | Regions become detailed on demand—not all at once. |
| Public mutation verbs | Dig, place, and related world edits are admitted only through the contract. |
| Public inspection | Bounded reads, queries, snapshots, telemetry, or events—never privileged storage paths. |
| Streaming lifecycle | Active and cold regions remain tractable; large worlds do not require permanent full residency. |
| Collision / occupancy truth | Contacts and occupancy queries read material authority, not disposable meshes. |
| Generic behavior extension | External systems observe authoritative matter and request admitted changes; their concepts, state, and rules remain external. |
| Dynamic material volumes | Movable, externally mutable voxel volumes share the same truth contracts as static world matter. |
| Persistence of scars | Edit deltas and related world-state scars persist without full-volume dumps. |
| Derived presentation | Surfaces and surface dressing rebuild from truth; derived geometry is not authority. |
| Object / clutter registration | Matter-backed assemblies (vegetation, micro-objects, similar) can register without baking into one terrain slab. |
| Content injection | Consumers drive how base volume is produced; the substrate does not own a generator. |
| Volume-general shape | Contracts do not force planetary heightmap or gravity-aligned terrain as the only legal world. |
| GPU-resident + async boundary | Authoritative matter can live GPU-resident; async GPU work does not break the consumer contract. |

Supporting principles that constrain *how* the product is consumed (not
implementation checklists):

- Consumers must not receive privileged access to internal voxel storage.
- Mutations enter through explicit public commands.
- Inspection uses bounded public interfaces.
- The same public boundary must serve validation harnesses and external game
  crates.
- Implementation ownership of work may move between CPU and GPU without
  changing the consumer contract.
- Vegetation and clutter presentation stays derived from matter—not a
  disconnected prop layer that desyncs from material truth.

---

## 7. Constraints and invariants

### Binding product constraints

**Authority: C-001, C-002, C-005, C-006, C-007, C-008, C-009, C-010, C-011, C-012, C-013, C-014, AD-001, AD-002, AD-003, AD-004, AD-005, AD-006, AD-007.**

- **Substrate first.** Moria is world infrastructure consumed as a Rust / Bevy
  crate ecosystem, not a finished visual game engine claimed before feasibility
  and visual-acceptance gates are met.
- **No privileged harness path.** Adjacent consumers—including any walkable
  validation harness—have no privileged access into the substrate.
- **Walkable harness optional for “done.”** A walkable-world harness may exist
  for validation; it does not define product identity or completion.
- **Everywhere mutation and first-class deep Z** are binding product outcomes.
  Deep Z means genuine volumetric depth, not heightmap terrain.
  Natural-looking presentation depends on the consumer’s world—coherent
  material presentation for that domain, not a single natural-overworld
  mandate.
- **Volume-general contracts** are required even though ship/station *content*
  is not current delivery or validation.
- **Dynamic voxel volumes** are a binding product capability; their behavior is
  not substrate policy.
- **Physics and damage are adjacent, not baked in.** Generic observation and
  admitted-mutation seams are required; behavior-specific concepts, state,
  fields, and rules remain external.
- **Generation is not substrate product.** Procedural / deterministic world
  generation is game-dependent and lives above the substrate.
- **GPU residency and an async-capable command/query boundary** are binding
  product direction and a deliberate distinction from CPU-driven voxel engines.

### Invariants (must remain true)

**Authority: C-003, C-004, C-005, C-006, C-007, C-008, C-009, C-010, C-011, C-012, C-016.**

1. **Matter is authority; views are disposable.** Presentation, dressing, and
   debug geometry never become truth.
2. **One consumer contract.** Harness and external game use the same public
   facade.
3. **No second world for collision.** Occupancy and collision truth are
   material, so plug-ins and consumers cannot invent a parallel mesh world.
4. **Mutation is universal within the contract.** Decorative-only solid
   geometry that cannot be edited under the same rules is not the product
   model for exposed material.
5. **Depth is volume, not paint.** Underground and freeform interiors are
   expressible as real material volume.
6. **Homogeneous emptiness and solid are cheap.** Scale depends on not paying
   full detail cost for untouched volume.
7. **Scars are first-class persistence.** Edits survive without full dumps.
8. **Residency does not break contracts.** GPU-resident work and async
   completion remain behind the public command/query boundary.
9. **Behavior systems are guests.** They attach through generic seams, own
   their state and rules, and do not own voxel storage.
10. **Content algorithms are guests.** Generators and fill strategies run on
    seams; they are not substrate law.

---

## 8. Validation principles

**Authority: C-003, C-004, C-005, C-006, C-010, C-013, C-015, C-016, AD-001, AD-002, AD-003, AD-007.**

How the product proves itself—without turning harness content into product
requirements.

- **Contracts over spectacle.** Evidence shows that inspection, mutation,
  streaming, collision truth, persistence, GPU-resident behavior, and (when
  exercised) generic behavior-extension seams hold—not that a particular
  forest postcard or character controller exists.
- **Same-interface proof.** Harnesses must use the public facade available to
  external games. Privileged internal paths invalidate the proof.
- **Mutation is the honesty test.** Dig and place (or equivalent admitted
  edits) must leave honest cut faces and rematerialized presentation from
  truth; a world that only looks good until edited has failed the material
  claim.
- **Collision against truth.** Movement and contact proofs read voxel matter,
  not disposable meshes.
- **Sparse scale is real.** Regions large enough that full raw-voxel residency
  is unreasonable must remain tractable under streaming and homogeneous cheap
  storage.
- **Deep volume is exercisable.** Depth content must be reachable as material
  volume (for example caves, strata, buried structure), not as skybox or
  painted floors.
- **Measurable substrate quality.** Benchmarks and evidence capture mutation
  response, streaming, GPU memory behavior, collision-truth honesty, and
  behavior-extension readiness when exercised—with machine context so results
  are comparable. Specific performance numbers, demo routes, and acceptance
  scenes belong to harness/TDD design, not this vision.
- **Optional behavior proof.** A simple external plug-in may demonstrate that
  it can observe truth and submit admitted changes. It must not cause the
  substrate to acquire physics, damage, or another behavior vocabulary.
- **Optional walkable harness.** A walkable third-person proof may make
  claims undeniable to humans; its controls, characters, assets, curated
  routes, content palettes, and generation pipelines are harness particulars,
  not substrate requirements.
- **No premature “finished engine” claim.** Do not claim a released, finished
  visual engine before feasibility and visual-acceptance gates are met.

---

## 9. Non-goals

**Authority: C-001, C-002, C-012, C-013, C-014, AD-001, AD-004, AD-006, AD-007.**

Explicitly out of product scope:

- Shipping a game, game mode, progression loop, or game-rules stack.
- Treating validation-harness content, third-person controllers, demo routes,
  content palettes, or machine-specific demo targets as substrate requirements
  or as mandatory delivery for product completion.
- **Baking deterministic or procedural world generation into the substrate** as
  product identity. Generation algorithms are game-dependent and run on top of
  the substrate; consumers own them.
- **Baking in physics, damage, or another behavior model** as product identity.
  The substrate exposes generic extension seams, not canonical strength,
  gravity, force, health, resistance, fracture, damage, or solver fields and
  rules. A proof plug-in is optional demonstration, not required delivery.
- Implementing excluded layers here: System / LLM, spells, gas / pricing
  policy, combat rules / AI behavior, agent labor, building UI / blueprints as
  gameplay, mechanisms as game entities.
- Delivering or specifically validating freeform ships, stations, or other
  multi-deck freeform hulls as current product work—those remain
  future-consumer examples that motivate volume-general contracts.
- Full fluid simulation and cellular automata (fire, wetness, growth) as
  current product requirements—these may appear as future consumer concepts or
  format hooks unless later selected explicitly.
- Tree felling or rigid-body conversion of vegetation as current product
  requirements (future consumer concepts unless later selected).
- Web / wasm as a Product One or substrate target platform.
- Claiming a released, finished visual engine before feasibility and
  visual-acceptance gates are met.
- Limiting substrate identity to a Minecraft-style cube aesthetic, a single
  natural-overworld content palette, static scenery without movable material
  volumes, or heightmap terrain that only pretends to have depth.
- Assuming gravity-aligned planetary terrain as the only legal world shape in
  substrate contracts.
- Structural integrity / cave-in simulation, span tables, fortress engineering
  UI, navigation graphs, multiplayer services, weather simulation, seasons,
  growth systems, and fine splash / particle matter layers as current substrate
  product (possible future consumer or plug-in concerns unless later selected).

---

## 10. Future consumers (context only)

**Authority: C-002, C-013, C-014, AD-001, AD-004.**

Reference material describes possible later products that motivate reusable
material-world capabilities. Their gameplay, characters, assets, content
palettes, and presentation are **not** current Moria scope. They illustrate
what the substrate must remain able to support:

- A System-driven ARPG on a continuous natural world.
- A fortress / colony game with engineering and designation play.
- A descent-style roguelike through deep geology.
- Pure sandboxes.
- Games whose players and enemies are voxel volumes that move and are changed
  under the same matter contracts as terrain, while external physics and damage
  systems own every behavioral rule and submit their results through admitted
  mutations.
- Freeform ship and station volumes (for example a single-captain trading and
  combat game where the hull is real material space rather than abstract slots).
  That fiction motivates everywhere mutation, first-class volumetric depth
  through multi-deck interiors, GPU-resident matter at combat and design scale,
  generic behavior-extension seams, and truth-vs-view so externally defined
  damage and salvage stay honest—**without** importing its fiction, UI, mission
  systems, freeform-hull *delivery*, or any behavior model into current Moria
  scope.

A “walkable world” proof shape (curated region, forest, ruin, dig-as-demo) may
be used by a validation consumer to make substrate claims undeniable. That
shape’s content, controls, milestones, performance tables, and curated
generation pipeline remain **context for what the substrate might support**,
not the definition of the product itself.

---

## 11. Resolved scope decisions

These human decisions are closed. Do not reopen them in product design without
an explicit new human decision.

| Topic | Decision |
| --- | --- |
| AD-001 — Walkable-world visual validation harness | **Adjacent artifact.** May exist for validation; does not define product identity or “done.” Moria is not limited to a Minecraft-style world. |
| AD-002 — Natural-looking terrain, everywhere mutation, first-class deep Z | **Everywhere mutation and first-class deep Z are binding.** Deep Z means genuine volumetric depth, not heightmap terrain. Natural-looking presentation **depends on the consumer’s world**—coherent material presentation for that domain, not a single natural-overworld mandate. |
| AD-003 — GPU-resident / asynchronous-GPU-capable architecture | **Yes, binding.** GPU residency is an important product feature: it enables many gameplay capabilities and is a core distinction between Moria and CPU-driven voxel engines. Specific simulations remain design-selected later. |
| AD-004 — Multi-world freeform volumes (ships / stations) | **Contracts are volume-general** (must not assume gravity-aligned planetary terrain only). **Delivering or specifically validating ships and stations is not current scope**; they remain future-consumer examples. |
| AD-005 — Dynamic voxel volumes | **Yes.** The substrate supports material volumes that move and can be mutated under the same truth contracts as static matter. Physics, damage, and other behavior that causes those changes remains external. |
| AD-006 — Deterministic / procedural world generation | **No.** Generation is an algorithm that runs on top of the substrate and is game-dependent; it must not be baked into substrate identity. |
| AD-007 — Physics and damage behavior | **Generic extension seams yes; behavior policy no.** External systems observe authoritative matter and request admitted changes. They own gravity, force, contact response, health, resistance, damage, fracture, and all related state and rules. Collision/occupancy **truth** against voxel matter remains a substrate concern so plug-ins and consumers share one material world. A proof plug-in is optional; no particular engine or behavior vocabulary is required. |

Older planning language that still titles the effort as “Product One — The
Walkable World” is superseded on identity by this product vision. Engineering
docs that list deterministic generation under the public facade describe
implementation or harness practice; generation is **not** substrate product
identity. Seed language that listed structural integrity, cave-ins, or rigid
conversion as non-current remains context for future consumers; this vision
requires only **generic behavior-extension seams** without importing a
substrate-owned physics or damage model, fortress span tables, or fortress UI.

---

## 12. Open product-boundary questions

None currently open that would change product identity, purpose, or boundary.

Engineering and milestone sequencing—including the generic extension
contract’s technical shape, presentation technique choices, storage layouts,
and platform engineering constraints—remain design and technical concerns, not
vision-identity blockers. This document intentionally does not decide them.

If a later design choice would reclassify substrate vs consumer ownership
(for example promoting full fluid CA, tree felling, structural collapse
simulation, freeform-hull delivery, or built-in physics or damage behavior into
substrate product), that requires an explicit human scope decision—not silent
expansion from seed implication.

---

## 13. Provenance (not a substitute for the vision above)

Source materials informed this synthesis. They are not required reading for a
designer once this document is approved. Authority among seeds: project
boundary first; GPU-resident architecture note as supporting principles
(residency and async boundary elevated to product direction by human decision);
broad voxel reference third; Product One seed last as validation example.
Conflicts resolve toward the boundary without silent expansion of scope.
Explicit human decisions override earlier vision wording and seed implication
when they reclassify substrate vs consumer ownership.

| Source | Contribution |
| --- | --- |
| Human-approved scope boundary (`docs/vision.md`) | Binding product identity, purpose, boundary, outcomes, non-goals, resolved Q1–Q7, and validation posture. |
| Project boundary seed | Reusable substrate as product; games separate; harnesses public-interface only; System / LLM / spell / gas / combat / AI / building layers out of scope. |
| GPU-resident substrate seed | Supporting principles: sparse residency direction, command/query/event boundary, no privileged storage access, derived meshes never truth; optional CA/integrity/particle extensions not current requirements. |
| Voxel-world substrate seed | Architecture reference motivating coherent domain presentation, everywhere mutation, deep Z, substrate-not-game layering, and GPU-resident direction—without importing generation pipelines, fluid tiers, integrity span tables, building verbs, nav, multiplayer, or game examples as requirements. |
| Product One seed | Downstream / validation example motivating fully material walkable proof, dig/place honesty, collision against truth, sparse streaming, scar persistence, and measurable quality—without importing third-person character, curated region postcard, content palette, performance tables, milestones, or generation pipeline into product identity. |
| System-substrate pivot notice | Excluded source; contributes no product requirements. Useful substrate-only principles live in the GPU-resident note. |

**Not elevated into product requirements:** crate structure, algorithms, brick
dimensions, voxel size, buffer or index formats, atomic widths, named graphics
backends, platform-specific engineering constraints, portability mechanisms,
task breakdowns, implementation milestones, example-game fiction, characters,
controls, assets, or validation-harness particulars—unless a later explicit
human decision makes a specific choice binding at product altitude.
