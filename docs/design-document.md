# Moria product design

Status: reconciled product baseline, September 6, 2026; not a technical design.
Authority: [seed](seeds/project-boundary.md) and
[decisions](product-design-decisions.md).

## Product and consumer

Moria is a reusable high-performance sparse material-volume engine for Rust and
Bevy consumers. Consumers supply content, inspect and change material, move
volumes, display results, and save them. Rendering and collision geometry refer
to the same material rather than independently maintained worlds.

The motivating space game includes authored ships, giant stations, asteroids,
debris fields and other bodies. Their rules are not Moria's rules. The initial
consumer is a simple asteroid demonstration, not that game.

## First-version requirements

This is a new requirement set. Historical REQ/TECH identifiers are not aliases
and do not add obligations to it.

### MOR-01 — Generic volumes and content (DEC-01)

Provide stable volume identity, local three-dimensional material coordinates,
and independent placement. Material and shape describe interior voids as well
as exterior and edited surfaces. No privileged up axis, terrain surface,
camera, player or ship type. Static and movable volumes use the same material
model. Consumers provide authored or generated base content.

### MOR-02 — Sparse GPU-oriented work (DEC-02)

Keep empty and homogeneous regions compact. Active detailed material and bulk
operations should stay on the GPU where practical. Bound residency according
to consumer needs and available resources rather than full world extent.
Avoid a mandatory full CPU material mirror or routine whole-world transfers.
CPU staging, persistence and bounded views are legitimate. Prove efficiency
with measurements rather than assuming GPU execution guarantees performance.

### MOR-03 — Editing and material consistency (DEC-01, DEC-05)

Support bounded inspection, addition/removal of material, and placement changes.
Submission and completion are distinct. Results identify the revision they
represent. Admitted work reports completion or failure; no silently partial
edit presented as success. Observers must not see half-published updates.
Commands account for relevant preconditions against authoritative material,
not merely a stale cached observation. Exact command and atomicity boundaries
belong to technical design and must be explicit, not implicitly whole-world.

### MOR-04 — Multiple CPU observation views (DEC-03)

Consumers define multiple independent regions and information selections.
Background jobs refresh read-only snapshots asynchronously, independently of
rendering. Reading an available snapshot does not require a fresh GPU query.
Expose coverage, represented revisions, readiness and failures. Unavailable
data is not empty material. Readers see complete snapshots, not partial copies.
Moving a request cannot relabel old data as covering its new region.

Views can move, resize and be released independently. Bound aggregate memory
and synchronization work. Overlapping requests may share storage or transfers,
but need not do so. Freshness lag is visible; instantaneous coherence is not
promised. Views support responsive observation, not mutation authority.

### MOR-05 — Derived presentation (DEC-01, DEC-07)

Display exterior and cavern surfaces from material truth, including newly cut
surfaces. Presentation may lag, but cannot redefine occupancy or collision.
Make material, voids and edits legible without requiring fancy game art.
Do not prescribe cube aesthetics, SDFs or a meshing method at product level.

### MOR-06 — Replaceable physics integration (DEC-04)

Support trusted modules with coordinated GPU material access, compute
scheduling, module-owned state/resources and material/placement changes. Do
not force GPU physics through CPU UI views or a closed catalogue of today's
anticipated effects. Preserve ownership and consistent publication, not
uncontrolled writes. A future repeatable physics module must be able to
coordinate its inputs and state with Moria's execution.

Storage, editing, persistence and presentation work without a particular
physics engine. Physics-specific behavior stays together rather than spreading
through the substrate. Prove the boundary with minimal collision functionality
and a minimal alternative module; replacing it must not require surgery across
the engine. This is not a promise of zero-adaptation third-party engine ports,
nor a requirement to implement full physics now.

### MOR-07 — Repeatability for testing (DEC-05, DEC-06)

Same starting material, supported configuration and ordered material inputs
produce repeatable material results. Name the software/backend configuration.
Tests must not randomly pass or fail due to ordering races. Reconstruct the
starting state and rerun inputs to demonstrate this; live rewind is unnecessary.
No cross-GPU or rendered-pixel identity requirement. Physics repeatability
requires cooperation from that module and is not implied by core results alone.

### MOR-08 — Persistence (DEC-01, DEC-07)

Save and restore material edits and placements. When saving differences from
base content, identify that base and reject mismatches rather than applying
edits to a different world. Consumers choose storage and own game/module state.
Report save/restore failures. Saving only derived meshes is insufficient.

### MOR-09 — Resource pressure and lifecycle (DEC-02, DEC-03)

Bound interest, views, queries, edits and transfers. Expose delays, unavailable
data or admission failures instead of unbounded allocation or fabricated empty
material. Consumers can release their requests. Outstanding GPU/CPU work must
not access reclaimed resources. Retain useful diagnostics for failures and
performance. Rendering failures do not silently alter material.

### MOR-10 — Minimal consumer demonstration (DEC-07)

Generate a fixed-seed asteroid with caverns, fly through it, inspect material
through CPU views and cut material while observing surface/collision updates.
Generator, camera and controls use ordinary public consumer APIs, not privileged
demo-only paths. Exercise simultaneous nearby and distant views without
requiring elaborate UI. No characters, weapons, ship systems or space-game
simulation. Save/reload and repeatability can be automated tests.

### MOR-11 — Early measured performance (DEC-08)

Establish a representative workload on named hardware early. Retain seed,
scale, detail, edit rate, view requests, settings and software identity. Measure
frame-time distribution/stalls, edit-to-visible latency, collision freshness,
CPU-view lag, memory, GPU work and CPU/GPU transfer volume. Include cold
movement, active edits and resource pressure, not only a warm stationary scene.

Agree concrete budgets from evidence before declaring performance accepted;
guard regressions on comparable workloads. If the architecture cannot deliver
responsive exploration/editing at useful scale, revise it rather than treating
correctness tests as proof of success. No inherited arbitrary numeric targets.

## Acceptance and deferred scope

The demo exercises the integrated product. Automated and real-GPU tests also
cover multiple-view lifecycle, pressure, repeatability, save/load, failures and
module replacement. Mock-only tests cannot prove real material-to-collision,
material-to-presentation or GPU-access plumbing. Performance requires retained
measurements and agreed budgets; it is not yet qualified.

Deferred: networking; cross-GPU bit identity; live rollback and its fixed window
or performance tier; coordinated participant rewind; full physics, heat,
forces, fracture, ship/core connectivity, debris classification, weapons, AI
and gameplay. Generation remains consumer code. Do not build a rollback
framework merely to leave room for future physics.

## Technical design generation

Bro V4 must generate the replacement TDD with adversarial review using the
current seed, decisions and this GDD. Its job is to propose and justify
implementation choices, expose unresolved feasibility questions, and trace
them to MOR requirements. The archived TDD is excluded from assignment inputs.
Do not convert its old queues, budgets, arithmetic or protocol details into
requirements by copying them. No issue decomposition until this design stage
has been exercised and reviewed.
