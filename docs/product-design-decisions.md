# Product decisions — September 6, 2026

Paraphrases of Jason's approved discussion, not verbatim quotations. This record
replaces the historical decisions for active planning.

| ID | Decision |
| --- | --- |
| DEC-01 | Generic material volumes, not terrain or ship-specific infrastructure. Ships, stations, asteroids and debris are consumer examples. |
| DEC-02 | Sparse GPU-oriented material processing for performance, not a ban on CPU coordination or bounded observation caches. |
| DEC-03 | Multiple independent CPU observation views with consumer-defined regions/information; asynchronous jobs maintain them; explicit coverage, readiness and freshness; shared resource limits; no rendering dependency. |
| DEC-04 | Deep, extensible, replaceable physics integration with GPU access. No particular physics engine dependency throughout Moria. Minimal integration proof now; full physics later. |
| DEC-05 | Repeatability for testing on a supported configuration. Future physics must cooperate; deterministic voxels cannot make arbitrary physics deterministic. |
| DEC-06 | Defer networking, cross-GPU deterministic qualification, live rollback, the minimum 20-tick window, coordinated participant rewind and its frame-time performance tier. |
| DEC-07 | Fixed-seed asteroid/cavern flythrough, CPU-view material inspection, and cutting with surface/collision updates. Demo generator and camera are consumer code. Automated save/reload and repeatability proof. |
| DEC-08 | Measure representative exploration/editing early, derive budgets from evidence and review, retain measurements and guard regressions. High performance is required, not a late optimization wish. |
| DEC-09 | No first-version weapons, heat propagation, forces, fracture, ship-core classification, debris rules or full game. Storage representation is an engineering decision. |
| DEC-10 | Reconcile seeds/GDD, retire old TDD, then generate a new TDD in Bro V4 with drafting and adversarial review agents. Do not handwrite its replacement. |

Reference hardware, workload scale, performance budgets and the exact minimal
collision demonstrator remain to be selected with engineering evidence.
These unknowns do not authorize restoring retired requirements.
