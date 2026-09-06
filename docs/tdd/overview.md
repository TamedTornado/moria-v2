# Moria technical design

Status: implementation-ready

Binding product contract: [`../design-document.md`](../design-document.md)

Supporting authority: [`../product-vision.md`](../product-vision.md) and
[`../product-design-decisions.md`](../product-design-decisions.md)

This directory is the complete technical contract for the clean Moria rebuild.
The product design decides what Moria is; this TDD decides how it is built.
Terms such as **must**, **rejects**, and **terminal** are testable implementation
requirements. A `TECH-###` identifier is stable and must not be reused or
renumbered if its contract survives a revision.

## Architecture at a glance

Moria is one Cargo package with a public `moria` library target and a
`moria-qualify` binary target. The binary is a separate Rust crate that imports
only the library's public API, so it proves the external-consumer boundary
without requiring a second package or workspace. The library is organized
internally as feature plugins and domain modules, but presents one facade. It
owns a versioned canonical state machine, GPU-resident sparse material roots,
bounded asynchronous work, collision truth, derived presentation, rollback,
replay, and checkpoint seams.

```text
consumer / moria-qualify
          |
          v
  MoriaPlugin + MoriaClient
          |
    bounded admission
          |
          v
 canonical tick coordinator <---- deterministic participants
          |
    prepare -> validate -> construct COW state -> hash -> publish
          |
          v
 GPU-resident canonical root (Bevy RenderDevice / RenderQueue)
    |                 |                    |
 queries/collision    rollback roots       derived presentation
    |                 |                    |
 bounded readback     replay/checkpoint    Bevy render entities
```

The authoritative sparse representation is a logical immutable radix tree of
8×8×8 material bricks. Each confirmed tick publishes a new root after all
commands and participant effects have passed deterministic validation.
Unchanged nodes and bricks are shared by retained rollback frontiers. Physical
GPU slots, cache locations, work completion, mesh order, and presentation are
not canonical.

The baseline implementation supports native Metal, Vulkan, and DX12 adapter
paths through Bevy 0.19 and its wgpu 29.0.3 renderer. Replay-grade authority
requires capability validation plus the canonical-math startup vectors; it
does not require a vendor/driver qualification record. Moria claims
same-machine replay only and has no cross-machine, cross-GPU, or multiplayer
determinism tier. Web and WebGL are not targets.

### TECH-001 — Product boundary and one facade

Implements: REQ-002, REQ-006, REQ-024, REQ-044

`moria` is reusable infrastructure, not a game. Its public API contains only
world configuration, content, material and volume registration, tick input,
participants, interest, query, observation, persistence, telemetry, and Bevy
integration. It contains no generator, player, camera, physics, damage,
networking, AI, economy, or gameplay vocabulary. `moria-qualify` is an ordinary
consumer crate and may use no `pub(crate)` API, internal GPU buffer, test-only
mutation hook, or feature that an external consumer cannot select.

### TECH-002 — Authority and ownership boundary

Implements: REQ-001, REQ-005, REQ-007, REQ-026

Only the canonical tick coordinator can replace the live canonical root.
Consumers own input bytes, base-content production, the semantics and opaque
representation of participant state, and checkpoint storage. Moria owns
lifecycle pins for immutable participant state tokens and copies declared
snapshot blobs plus reconstructible participants' required replay-record blobs
into that checkpoint storage. Moria also owns admitted copies of input,
material truth,
physical GPU allocation, revision publication, canonical outcomes, and derived
work. Consumer-registered checkpoint stores, bundled-content stores, and the
replay sink are selected by stable IDs and receive only bounded
owned values/completion tokens. Public replay consumes owned records in a
private builder and can publish only its fully verified final frontier. A
correction durably appends one bounded branch record before atomically
replacing the active in-memory suffix and live frontier; superseded append-only
bytes remain diagnostic evidence but are not part of the active semantic log.
A successful queue operation means admission only. Authority changes only in
the coordinated publication transaction reported by a confirmed tick or a
successful construction/correction/restore/recovery receipt.

The CPU may retain bounded configuration, stable identities, receipt state,
root handles, tick/replay logs, participant commitments, persistence staging,
and query results. It must not retain a dense or sparse full CPU mirror of
canonical voxel cells. No presentation or cache state may enter a tick input,
canonical decision, or hash.

### TECH-003 — Workspace and dependency boundary

Implements: REQ-002, REQ-007, REQ-023, REQ-024

The clean repository has this shape:

```text
Cargo.toml
Cargo.lock
rust-toolchain.toml
AGENTS.md
assets/
  shaders/
    canonical/
      math/
    presentation/
fixtures/
  replay/
src/
  lib.rs
  prelude.rs
  config/
  facade/
  canonical/
    math/
  content/
  storage/
  runtime/
  query/
  collision/
  participant/
  persistence/
  presentation/
  telemetry/
  bevy/
  bin/
    moria-qualify/
      main.rs
      cli/
      evidence/
      oracle/
      scenarios/
tests/
docs/tdd/
```

The root manifest defines `[lib] name = "moria"` and one `[[bin]]` named
`moria-qualify`; it does not declare a Cargo workspace. This is the smallest
package boundary that produces both required artifacts. The binary crate uses
`use moria::...` exactly as an external package would and has no shared private
module tree with the library.

`moria` uses Bevy `=0.19.0`; any direct wgpu dependency uses `=29.0.3`,
matching Bevy. `Cargo.lock` is committed. The Rust toolchain is pinned in
`rust-toolchain.toml`; changing a canonical contract, math table, shader, or
encoding changes the configuration/replay identity. Rust, Bevy, wgpu, Naga,
adapter, and driver versions remain evidence context, not per-driver
qualification keys.

`moria` exposes no wgpu type from its general facade. The deliberately coupled
GPU-participant API is under `moria::bevy::gpu_participant` and may expose only
the adapter types named in [interfaces.md](interfaces.md). Default Cargo
features are `bevy`, `persistence-zstd`, and `presentation`; canonical
semantics do not vary by feature. Optional features may remove derived
capabilities or stores, never change canonical bytes for the same declared
contract.

## Document map

- [architecture.md](architecture.md): ownership, canonical state, sparse
  representation, transition, hashing, and rollback roots.
- [interfaces.md](interfaces.md): public Rust facade, inputs, receipts,
  complete callable surface, resource-budget schema,
  queries, participants, observations, errors, and state machines.
- [gpu-runtime.md](gpu-runtime.md): Bevy/wgpu integration, WGSL ABI,
  scheduling, pools, bounds, device loss, and portability.
- [content-persistence.md](content-persistence.md): base content, scars,
  durable checkpoints, restore, replay, and reclamation.
- [collision-presentation.md](collision-presentation.md): canonical collision,
  participant artifacts, meshing, dressing, and revision isolation.
- [validation.md](validation.md): automated, real-GPU, same-machine replay,
  persistence, rollback, performance, and evidence obligations.
- [traceability.md](traceability.md): requirement-to-technical-contract index.
- [decisions.md](decisions.md): durable human-review feedback and its applied
  technical interpretation.

## Cross-cutting invariants

1. A world has one live `CanonicalRoot`; only a confirmed tick or exact
   retained-frontier restore can replace it.
2. Tick zero observes the explicit pre-tick genesis state; tick `n > 0`
   observes confirmed `State[n - 1]`. Each uses one sealed `TickBatch[n]` and
   participant products bound to that source state. Timing cannot add or
   remove input.
3. One matter command is wholly applied at one volume revision or has no
   committed effect.
4. Unknown, cold, failed, or mismatched content is never encoded as empty.
5. Stable IDs, ordering, allocation outcomes, revisions, hashes, and canonical
   collision bytes are independent of physical slots and execution order.
6. Every authoritative output is integer/fixed-point and has one canonical
   byte encoding. Placement uses the world's genesis-bound fractional split,
   cell extent, and simulation-unit identity through `moria-fixed-v1`;
   participant-owned quantities declare separate representation contracts.
7. A root remains physically live while referenced by the live frontier, a
   rollback snapshot, a query, checkpoint, GPU submission, or active replay.
8. A derived artifact carries its source tick, root hash, and volume revisions;
   a mismatch makes it stale or discarded, never authoritative.
9. Every queue, allocation, output, readback, observation stream, retained
   window, and replay is bounded.
10. Backend or participant failure cannot partially publish a tick.
11. Participant coordination is one-phase: no same-tick participant DAG or
    handoff exists; bounded opaque events reach the owning tick receipt only
    after confirmation, and effects use ordinary canonical command ordering.

Performance acceptance is correctness-first. The superseded `P1`–`P10` gates
are not claims of this TDD; TECH-067 defines the one named rollback tier and
TECH-068 requires hardware-contextual measurements for every retained hot path.

## Engineering sequence

Implementation proceeds by vertical proof, not by building presentation first:

1. canonical encoding, arithmetic, CPU oracle, IDs, and tick ordering;
2. public facade with bounded headless state-machine tests;
3. real-GPU brick lookup, copy-on-write mutation, readback, and root hashing;
4. rollback roots, replay, participants, and persistence;
5. collision and GPU-participant artifact paths;
6. presentation and dressing;
7. same-machine replay/contamination evidence and performance receipts.

Each stage must retain the same public path. A CPU oracle is test evidence, not
an authority backend or runtime fallback.

### TECH-004 — Repository implementation contract

Implements: REQ-007, REQ-021, REQ-023, REQ-036, REQ-044

The root `AGENTS.md` must contain the following project-specific instructions
(wording may gain clarifications, but commands and constraints are normative):

```markdown
# Moria agent instructions

## Commands

- Format (write): `cargo fmt --all`
- Format check: `cargo fmt --all -- --check`
- Check: `cargo check --all-targets --all-features`
- Lint: `cargo clippy --all-targets --all-features -- -D warnings`
- Test: `cargo test --all-targets --all-features`
- Shader validation: `cargo run --bin moria-qualify -- shaders validate`
- Replay determinism: `cargo run --bin moria-qualify -- replay verify --fixture fixtures/replay/core-v1 --runs 8 --evidence target/moria-evidence/replay`
- Build: `cargo build --all-targets --all-features`
- Development scenario: `cargo run --bin moria-qualify -- scenario public-boundary --mode candidate --evidence target/moria-evidence/dev`
- Full local gate: run format check, check, lint, test, shader validation,
  replay determinism, then the development scenario in that order.

Replay determinism runs on the current machine and must use a real GPU for a
replay-grade result. `UNAVAILABLE` is not `PASS`. There is no cross-vendor
matrix or driver requalification command.

## Module and naming rules

- Keep `lib.rs`, `prelude.rs`, plugin `mod.rs` files, and the evidence
  binary entry point as wiring/facades.
- Organize by domain responsibility. Do not add root catch-alls named
  `types.rs`, `systems.rs`, `components.rs`, `resources.rs`, or `utils.rs`.
- A feature module owns its Bevy plugin wiring, systems, resources, messages,
  tests, and shader ABI.
- Public types use descriptive nouns. Fallible actions use verbs. State enums
  end in `State`, receipts in `Receipt`, stable identities in `Id`, immutable
  root references in `Root`, and configuration limits in `Limit` or `Budget`.
- Canonical wire structs end in `Wire`, are fixed-width, and live beside their
  WGSL ABI. Host-only structs must not be cast to a wire type implicitly.
- New public APIs require rustdoc examples and an error/bounds contract.

## Dependency rules

- The root package contains the public `moria` library crate and the
  `moria-qualify` binary crate. Do not introduce a Cargo workspace, another
  package, or another target without a concrete compile, reuse, or deliverable
  boundary recorded in the TDD.
- Pin Bevy and direct wgpu/Naga dependencies exactly. Do not create a second
  wgpu device in the Bevy path.
- Keep runtime/executor types out of the general public API. Do not add Tokio
  to the public contract; use pollable receipts and Bevy-driven progress.
- New dependencies need a written reason, compatible license, default-feature
  review, wasm assumptions review, and `cargo deny` policy update when that
  gate is introduced.
- Unsafe Rust is forbidden unless a reviewed module-level safety contract and
  Miri/test evidence are added. No unsafe code is needed for the baseline.

## Moria constraints

- No canonical state change outside a sealed numbered tick; genesis and exact
  retained-root restore are the only stated exceptions.
- No floating point, unordered map iteration, race-winner allocation, wall
  clock, OS entropy, callback order, or presentation readiness in canonical
  transitions.
- No unbounded channel, vector, retry loop, GPU probe, dispatch, readback, or
  retained history.
- Never expose internal storage buffers. GPU participants receive bounded
  source-hash-bound artifacts and fixed-capacity effect sinks.
- Submission is not completion. Do not recycle a GPU slot until its last queue
  use and mapping/decoding are complete.
- Treat unknown matter as unavailable, never empty.
- Physical slot IDs are not stable identities and never enter persistence.
- Every WGSL bounds check and overflow flag is part of the matching Rust ABI
  test. Pop every wgpu error scope.
- Implement the full TECH-070 facade; do not leave a prose-only capability or
  add a second callable path. Admission rejection returns owned requests, and
  every producer output reserves its TECH-036 count/byte capacity first.
- Observation filtering uses append-time filter facts, never current placement.
  Participant coordination remains one-phase with no same-tick DAG/handoff;
  opaque participant events are delivered only by confirmed tick receipts.
- Tests and tools use the public facade. CPU-oracle, mock, software-adapter,
  shader-compile, and rendered-frame evidence cannot be labeled real-GPU
  replay evidence.
- Canonical placement math lives only in `src/canonical/math/` and matching
  generated WGSL. Do not use floats, implicit numeric conversions, libm, or
  shader transcendental builtins in canonical code. Preserve the world
  placement format and participant representation-contract boundary.
- Preserve existing `TECH-###` meanings and `Implements:` requirement links.
  `TECH-063` is retired and must never be recreated or reused.
```

The initial `rust-toolchain.toml` must include `rustfmt` and `clippy`.
CI repeats format/check/lint/test/shader validation on Linux, provisions a
software Vulkan adapter for the candidate development scenario, runs
documentation tests, and reports replay verification as `UNAVAILABLE` when no
real GPU is attached. Dedicated same-machine runners execute the replay
command repeatedly on their current adapter. No result is compared across
vendors or promoted into a cross-machine qualification claim.

## Open Human Questions

None. The approved design explicitly leaves the consequential representation,
portability, and workload parameters to technical design, and this TDD selects
them without changing the product boundary.
