# Moria — clean restart

Reusable voxel-world infrastructure for Rust and Bevy consumers.

This repository starts again from documents, with a fresh Git history. No
previous implementation, assets, tests, generated issues, PRs, branches or
execution state have been adopted. Bro V4 is the intended execution system;
do not resume a Bro V1 run against this repository.

## Retained planning input

- [Seed documents and their authority order](docs/seeds/README.md).
- [Product design / GDD](docs/design-document.md).
- [Technical design / TDD](docs/tdd/overview.md).
- [Product vision](docs/product-vision.md) and
  [product design decisions](docs/product-design-decisions.md), explicitly cited
  by the TDD as supporting authority.

These files were copied byte-for-byte from `TamedTornado/moria` at commit
`c07eb8ae838cf85e886fe3288a3f1efefd9edef5` on September 6, 2026. The old repository
is retained separately as history; it is not this restart's work queue.

Retaining existing design documents does not assert that their previous
implementation or qualification succeeded. Review and test the planning inputs
through the new workflow before treating them as an executable issue backlog.
Downstream game examples in the seeds do not override the substrate-only product
boundary. No generated `issues.json` or old tracker mapping is carried forward.
