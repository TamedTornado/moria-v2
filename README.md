# Moria — clean restart

Reusable voxel-world infrastructure for Rust and Bevy consumers.

This repository starts again from documents, with a fresh Git history. No
previous implementation, assets, tests, generated issues, PRs, branches or
execution state have been adopted. Bro V4 is the intended execution system;
do not resume a Bro V1 run against this repository.

## Current planning input

- [Seed documents and their authority order](docs/seeds/README.md).
- [Product design / GDD](docs/design-document.md).
- [Current product decisions](docs/product-design-decisions.md).
- [TDD generation status](docs/tdd/README.md): pending a Bro V4 drafting and
  adversarial-review run; no replacement TDD has been handwritten.
- [Historical specifications](docs/history/pre-2026-09-06-reconciliation/README.md)
  are excluded from current planning inputs.

The initial documents were copied byte-for-byte from `TamedTornado/moria` at commit
`c07eb8ae838cf85e886fe3288a3f1efefd9edef5` on September 6, 2026. The old repository
is retained separately as history; it is not this restart's work queue. Current
seed/GDD/decisions were reconciled on September 6; the old specifications are
preserved in the historical directory rather than merged into the new scope.

Retaining existing design documents does not assert that their previous
implementation or qualification succeeded. Review and test the planning inputs
through the new workflow before treating them as an executable issue backlog.
Downstream game examples in the seeds do not override the substrate-only product
boundary. No generated `issues.json` or old tracker mapping is carried forward.
