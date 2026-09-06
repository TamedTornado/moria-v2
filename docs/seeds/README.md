# Moria seed documents

This directory contains the source material and binding clarification used to
plan Moria. The documents do not have equal authority, and their presence
together does not make every described capability a Moria deliverable.

## Authority order

1. [`project-boundary.md`](project-boundary.md) is the binding product target.
   Moria is a reusable voxel-world substrate consumed through public Rust APIs.
2. [`gpu-resident-substrate.md`](gpu-resident-substrate.md) is a supporting
   architecture note extracted from the original System pivot. It preserves
   only substrate-relevant principles and does not add features to the current
   milestone.
3. [`voxel-world-substrate.md`](voxel-world-substrate.md) is a broad supporting
   architecture reference. Its game examples and future extension ideas are
   context, not requirements.
4. [`product-one-seed.md`](product-one-seed.md) is a downstream consumer and
   validation example. Its third-person character, curated route, forest
   population, and machine-specific performance targets are not Moria
   requirements.

When documents conflict, [`project-boundary.md`](project-boundary.md) wins. A
supporting document becomes binding only when the project boundary or an
explicit human decision selects one of its claims.

## Provenance

The three original source files were provided on 2026-07-13:

- `product-one-seed.md`
- [`system-substrate-pivot.md`](system-substrate-pivot.md) (excluded-source
  notice; the original game/System document is not reproduced)
- `voxel-world-substrate.md`

The original System pivot mixed substrate architecture with downstream game,
LLM, spell, gas, combat, and AI design. It was intentionally excluded after the
product boundary was clarified, but the retained voxel document still contained
dangling references to it. `gpu-resident-substrate.md` now carries the useful
substrate-only material without restoring the excluded product scope.

The original System pivot remains outside this repository. Its SHA-256 and
exclusion rationale are retained in the excluded-source notice.
