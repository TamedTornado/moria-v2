# Moria product boundary

Current authority: Jason's September 6, 2026 product reconciliation.

Moria is reusable, high-performance sparse voxel-world infrastructure for Rust
and Bevy consumers. It owns material volumes and the operations needed to
inspect, edit, move, display, and persist them. It is not a game, a heightmap
system, or a collection of ship-specific components.

The motivating game has authored voxel ships, giant stations, giant asteroids,
debris fields, and other bodies. Conduits, reactors, weapons, heat, forces,
fracture, and ship ownership belong to consumers or replaceable simulation
modules. These explain why extensibility matters, not what to implement now.

Keep active detailed material and bulk work GPU-oriented, with sparse storage
for empty and homogeneous regions. Consumers can register multiple bounded,
read-only CPU observation views refreshed asynchronously and independently of
rendering. There is no mandatory full CPU voxel mirror.

Physics is deferred implementation, not an external-only restriction. Support
deeply integrated, replaceable physics modules, including GPU compute and
material access. Moria must remain usable without a particular physics engine.
Establish and test the boundary with minimal functionality, potentially
collision-only; do not build full physics now.

Require repeatable material results from the same starting state and ordered
inputs on a supported configuration, for reliable testing. Make participation
in repeatable simulation possible for future physics modules. Networking,
cross-GPU bit identity, live rollback, a 20-tick rollback window and its
restore-and-replay performance target are deferred, not first-version gates.

The initial demonstration is a fixed-seed asteroid with caverns and a
flythrough, material inspection and cutting. Its generator and camera are
ordinary consumer code. Save/reload and repeatability can be automated tests.
The consumer boundary is mandatory; precise package layout belongs to the TDD.

Excellent performance is a first-version requirement. Establish representative
workloads on named hardware early, measure responsiveness and resource use,
agree evidence-based budgets and guard regressions. Do not leave performance
validation until the end or invent universal numeric targets without evidence.

Voxel representation, SDF use, brick dimensions, copying algorithms and module
ABI details are technical decisions, not additional product requirements.
