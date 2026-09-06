# GPU-resident voxel substrate architecture

## Status and authority

This is a supporting architecture reference for Moria. It extracts the
substrate-relevant material identified by the
[`system-substrate-pivot.md`](system-substrate-pivot.md) excluded-source notice
without importing that original document's game, System, LLM, spell, gas,
combat, or AI scope.

[`project-boundary.md`](project-boundary.md) remains authoritative. The
principles below constrain a feature only when a design or technical decision
explicitly selects that feature for a Moria milestone. They are not an
implementation checklist.

## Sparse voxel storage direction

- Use a bounded top-level spatial index over a sparse brick pool rather than an
  unbounded film-oriented VDB tree.
- Represent homogeneous regions compactly so untouched air and solid material
  do not require fully allocated voxel bricks.
- Keep coordinates and GPU-visible allocation indices portable across Metal,
  Vulkan, and Direct3D; GPU-visible counters and atomics remain 32-bit.
- Treat rendered meshes as derived views. Voxel state and public substrate
  operations remain authoritative.

The exact brick dimensions, allocator, residency policy, and CPU/GPU ownership
belong to the technical design and must be justified by measurements.

## Command, query, and event boundary

Consumers must not receive privileged access to internal voxel storage.

- Mutations enter through explicit public commands.
- Inspection uses bounded public queries, snapshots, telemetry, or events.
- GPU work may complete asynchronously; consumers must not depend on direct
  buffer access or synchronous readback.
- Derived render data is never saved as world truth.
- The same public boundary must serve the validation harness and an external
  game crate.

This boundary preserves portability and allows implementation ownership to move
between CPU and GPU without changing the consumer contract.

## Optional future extensions

The original pivot also sketched coarse cellular simulation, structural
integrity, dynamic particles, and conversion between moving matter and static
voxels. Those are possible downstream extensions, not current Moria
requirements. They require separate scope decisions, acceptance criteria, and
measured feasibility before implementation.

Specifically, this document does **not** require:

- cellular automata, fire, fluids, granular settling, or structural collapse;
- PBD particles, debris, rigid conversion, or re-voxelization;
- Lua or another scripting runtime;
- LLM-authored code or WGSL kernels;
- gas metering or a game economy;
- player, combat, monster, spell, AI, or camera systems.
