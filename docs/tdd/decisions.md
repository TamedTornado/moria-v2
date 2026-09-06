# Technical design decisions

This record preserves human feedback received during technical-design review
and the technical interpretation applied to the TDD. Verbatim feedback is kept
separate from interpretation. Product authority remains in
[`../design-document.md`](../design-document.md).

## Human review entry

### Verbatim feedback

```text
Is this as simple as it can be while still satisfying the requirements? If yes, leave the TDD unchanged. If no, revise the TDD to make it the simplest sufficient design.
```

### Technical decision or clarification

The TDD was not yet as simple as it could be. The two-package Cargo workspace
did not provide a necessary isolation boundary: one Cargo package can build a
public `moria` library crate and a separate `moria-qualify` binary crate. Rust's
target privacy still prevents that binary from accessing the library's private
or `pub(crate)` implementation, so it remains an external-style consumer of
the public facade.

The repository contract is therefore simplified to one root package with one
library target and one qualification binary target. No canonical, GPU,
rollback, persistence, collision, participant, or validation mechanism is
removed: those mechanisms are the minimum selected implementation of explicit
approved requirements rather than optional product scope.

### Unresolved question

None.

---

## Human review entry — consumer-contract completeness

### Verbatim feedback

```text
TamedTornado (COMMENTED):
The regenerated TDD is substantially cleaner, and its determinism, rollback,
hashing, collision, persistence, and scope boundaries should be preserved.
However, comparison with the approved pre-amendment TDD found that the size
reduction also removed implementation-critical contracts. Please revise the
current TDD to close the following holes without restoring the old TDD
wholesale or reviving its over-engineered behavior scheduler.

1. **Complete the public facade.** `interfaces.md` says its Rust signatures are
   normative, but `MoriaClient` currently exposes only construction and tick
   submission. Add normative callable shapes, ownership, results, and receipt
   behavior for queries, interest upsert/withdrawal, observation subscription
   and polling, checkpoint requests, correction/restore, telemetry, and
   shutdown. The existing prose for these capabilities is not a substitute for
   a callable contract.

2. **Define `ResourceBudgets` completely.** It is named in `MoriaConfig` but has
   no normative field schema. Specify every bounded queue/pool/output retained
   by the current design, its portable maximum or configuration rule, and the
   cross-limit validation performed before genesis. In particular, make
   callback outputs, observation records and payload bytes, identity/lifetime
   records, query/readback resources, presentation work, checkpoint staging,
   rollback retention, and participant state/effects/snapshots unambiguous.

3. **Close base-content callback ownership.** Define
   `BaseBrickCompletion` (or its replacement) so capacity and worst-case bytes
   are reserved before consumer code runs, successful output lands in a
   Moria-owned bounded sink, and no unbounded consumer-owned collection or
   diagnostic allocation crosses into Moria. Define descriptor ownership,
   completion cardinality, cancellation, duplicate/late completion, exact byte
   validation, and resource release.

4. **Complete observation subscription semantics.** Define the subscription
   request and creation API, accepted finite volume membership, kind/spatial
   filtering, cursor start, close/drop behavior, gap recovery, bounded
   resnapshot, and resume behavior. Filtering must remain historically honest
   after movement, retirement, and directory-version reclamation. Either retain
   append-time filtering evidence or specify an equally exact replacement; do
   not evaluate historical facts against current placement.

5. **Connect lifecycle prose to exact APIs.** For every retained asynchronous
   operation family, define admission, cancellation point, pollable receipt
   states, terminal retention, retry rules, ownership returned on rejection,
   shutdown behavior, and device-generation behavior. Queries, observations,
   checkpoints, and correction/restore currently need particular attention.

6. **Make participant simplification explicit.** Do not restore the old
   ordering DAG, handoff buffers, conflict machinery, or prior-feedback ABI by
   default. Instead, state explicitly whether v1 supports or rejects:
   same-tick dependency between participants, bounded participant-owned event
   output to the consumer, and conflicts between participant effects. Ensure
   the selected minimal contract is sufficient for an external GPU-oriented
   physics/damage implementation to receive tick input, produce admitted
   effects, preserve its own state, and expose its own collision/simulation
   events without giving Moria a behavior vocabulary.

7. **State the performance-policy replacement.** The old TDD used P1-P10 hard
   architecture-feasibility gates. The current GDD appears to replace those
   with universal correctness, one named rollback qualification tier, and
   hardware-contextual benchmark reporting. Record that replacement explicitly
   and ensure the benchmark suite still measures sparse residency, mutation,
   query/collision, materialization, presentation, participant, checkpoint, and
   rollback paths. Do not silently imply the old gates remain satisfied.

8. **Mechanically clean the normative snippets.** For example,
   `MinimumVolumeRevision` currently declares `revision` twice. Inspect all
   normative Rust shapes for duplicate fields, unresolved named types, and
   missing facade connections.

After revision, update traceability and the review findings so approval asserts
both GDD coverage and implementation completeness. The desired result is the
current compact architecture with a complete consumer contract—not a return to
the 11,000-line design.
```

### Technical decision or clarification

The compact one-package architecture and the existing deterministic
publication, rollback, hashing, collision, persistence, and scope contracts
remain selected. The review requires implementation closure, not restoration
of the superseded large TDD.

The applied technical decisions are:

- add one complete normative facade surface and concrete receipt/cancellation
  behavior while retaining focused ownership contracts for each operation;
- make `ResourceBudgets` a closed nested schema with defaults, portable maxima,
  overload outcomes, and checked pre-genesis cross-limit equations;
- make base-source and store callback outputs producer-written into
  pre-reserved Moria-owned bounded completion cells;
- use a finite admitted observation membership and append-time immutable
  filter facts, with count/byte gaps and bounded frontier resnapshot/resume;
- retain a one-phase participant model: same-tick participant dependencies are
  rejected, bounded opaque participant events are supported for consumer
  delivery after confirmation, and effects use only ordinary deterministic
  command ordering/preconditions—no DAG, handoff, conflict subsystem, or
  prior-feedback ABI;
- explicitly supersede `P1`–`P10` with universal correctness gates, the one
  named 20-tick rollback tier, and hardware-contextual receipts covering all
  named performance paths; and
- require mechanical public-type/facade closure and distinguish approved GDD
  coverage from approved implementation completeness in the final gate.

### Unresolved question

None. The feedback resolves the consequential direction, and the remaining
field limits, lifecycle choices, participant event transport, and validation
mechanics are ordinary engineering decisions selected in the TDD.

---

## Human review entry — determinism addendum

### Verbatim feedback

```text
TamedTornado (COMMENTED):
Addendum to the determinism amendment (from Fable, confirmed by Jason):

1. Drop the cross-machine determinism tier. Remove from scope: the cross-GPU qualification matrix, per-driver requalification, the cross-vendor CI gate, the Metal/Vulkan cross-vendor kernel spike, and the DX12 qualification bookkeeping. Moria makes no cross-machine determinism claim until a conformance fixture exists someday. Multiplayer is not a product goal.

2. Keep replay-grade determinism as mandatory core, unchanged: canonical tick transitions, event-sourced mutation, the rollback ring, hierarchical hashing, replay artifacts. Replay determinism means: same machine, same genesis, same TickBatch stream, bit-identical hash sequence, every run.

3. Keep the full kernel contamination audit. Order-dependent atomics, race-produced bytes, and arrival-order compaction break same-machine run-to-run replay, not just cross-machine agreement. Only the cross-vendor motivation is dropped; the audit itself is unchanged.

4. Keep fixed-point simulation representation, with the justification updated: it makes replay and hash fixtures portable across CI machines and agent worktree runners, and immune to GPU driver updates. With f32 authoritative state, every golden replay file is pinned to one GPU-plus-driver configuration, which breaks the verification method itself.

5. Moria specifies a canonical parameterized fixed-point representation for simulation placement; fractional split and simulation unit are per-world genesis constants included in the configuration fingerprint; the canonical math library is generic over the split; no other physical quantities are defined by the substrate—deterministic participants declare their own representations under the existing participation contract.

6. New named component: canonical fixed-point math library. Multiplication with 64-bit intermediates and a specified rounding rule, division, sqrt, and the trig required for orientation composition—bit-identical CPU and GPU implementations. Verify by differential testing against an arbitrary-precision reference over generated cases. Use distinct types at the simulation boundary; no implicit conversion to or from f32.
```

### Technical decision or clarification

The TDD retires the former `TECH-063` cross-backend qualification matrix and
does not reuse that ID. It removes vendor/driver qualification manifests,
cross-vendor CI comparison, and backend-family conformance bookkeeping.
Metal, Vulkan, and DX12 remain supported runtime adapter paths, but their
identity is diagnostic and benchmark context only. Replay-grade authority now
means exact repeated execution on the same physical machine with the same
genesis and ordered `TickBatch` bytes; no networking or multiplayer behavior
is introduced.

Canonical ticks, event-sourced mutation, copy-on-write rollback roots,
hierarchical hashes, replay/export/divergence artifacts, and the complete
kernel-contamination audit remain mandatory. The audit explicitly rejects
race-produced bytes, atomic-winner authority, physical/arrival order, and
order-dependent compaction because each can violate same-machine replay.

`TECH-007` now freezes a per-world placement fractional split, exact cell
extent in raw simulation-unit increments, and consumer-defined simulation-unit
identity into genesis and the configuration fingerprint. `TECH-071` is the
new named `moria-fixed-v1` component: generic CPU/WGSL fixed-point operations,
specified ties-to-even multiply/divide/square-root reductions, canonical
CORDIC trigonometry for axis-angle orientation construction, distinct
simulation types, and no implicit float conversion. Participants declare
their own non-placement physical representations through bounded,
genesis-committed representation-contract descriptors.

### Unresolved question

None. A future cross-machine claim would require a new conformance fixture and
human-authorized technical contract; it is not guessed or reserved here.
