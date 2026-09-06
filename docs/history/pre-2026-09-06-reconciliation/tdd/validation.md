# Validation and evidence plan

Validation uses the same public facade as external consumers and layers
evidence so a host mock, compiled shader, rendered frame, or self-reported pass
cannot stand in for canonical GPU truth.

## Test architecture

### TECH-059 — CPU oracle and property tests

Implements: REQ-007, REQ-021, REQ-023, REQ-036, REQ-038

`moria-qualify` contains an independent, deliberately small CPU reference
model for canonical encoding, parameterized fixed-point arithmetic, cell
edits, sorted tick outcomes, collision primitives, radix commitments, and
replay hashes. It shares public wire definitions but no transition/storage
implementation with Moria. It is evidence only and cannot publish an
authority world.

Required deterministic tests include:

- golden bytes and digests for every canonical record and hash domain;
- arbitrary-precision differential cases for every `moria-fixed-v1` placement
  split `0..=16`: add/subtract, 64-bit-intermediate multiply, division, square
  root, narrowing, floor shift/division, ties-even half cases, overflow, and
  every quadrant and exact boundary class of `TurnQ32` plus generated samples
  across its full `u32` domain; the checked-in gain/arctangent table is
  regenerated at 256-bit precision and must byte-match the Rust/WGSL source
  table. Retained CPU and WGSL CORDIC golden vectors include raw turns
  `0x0000_0000`, `0x2000_0000`, `0x4000_0000`, `0x6000_0000`,
  `0x8000_0000`, `0xa000_0000`, `0xc000_0000`, `0xe000_0000`, and
  `0xffff_ffff`, plus one word on both sides of every quadrant center and
  reduction midpoint. They compare every iteration's `(x,y,z)` bytes as well
  as final `(sin,cos)`, so midpoint ownership, zero-residual branching,
  simultaneous updates, floor shifts, final rounding, and quadrant remap are
  independently fixed;
- orientation edges for exact square-root quaternion normalization (including
  `(1,1,0,0)`), composition/inverse, quantized-unit-shell membership,
  rational quaternion-vector rotation/transpose inverse and orthogonality,
  checked i64/i128 helper parity, and the maximum-radius displacement proof;
  generated accepted-quaternion cases assert the shell after every
  registration/composition and the retained proof checks all permitted
  rounding branches. Axis-angle goldens cover zero, positive/negative basis,
  diagonal, `i32::MIN`, and maximum nonzero axes at zero, half, and full-turn
  aliases, both odd half-angle truncation cases, and maximum turn word.
  A zero axis returns `ZeroAxis` even at zero/full-turn identity; injected
  checked-helper corruption returns `UnrepresentableAxis`, and CPU/WGSL wire
  tags and quaternion bytes match;
- generated edit state machines with create/retire/move/erase/place/patch,
  overlaps, stale preconditions, revision exhaustion, and failed atomic
  commands;
- generated sparse maps compared after every edit, compaction, checkpoint
  round trip, and rollback;
- hash sensitivity for matter, placement, IDs/allocator, simulation domain,
  participant/RNG commitment, and insensitivity to derived cache mutations;
- collision properties: no fact targets empty matter, all returned facts carry
  the source frontier and revision, and CPU/GPU parity for every
  `moria-collision-v1` formula. Golden edge fixtures cover closed-low/open-high
  points, face/edge/corner touches, inside witnesses, zero radius/extent/
  capsule/sweep, transform round trips, quaternion half-step rounding,
  exact world-space contact point/normal bytes for rotated and translated
  dynamic volumes, directed-normal preservation and post-rotation
  renormalization,
  parallel slabs, all 15 SAT axes and zero cross axes, equal penetration ties,
  capsule breakpoint/clamp regions, singular 2×2 sweep systems, irrational
  quadratic TOI floors, support-vertex ties, and every arithmetic failure;
- decoder fuzzing for truncation, trailing bytes, invalid tags/counts, zip
  bombs, corrupt hashes, and allocation overflow.

Schedule-perturbation tests submit identical sealed bytes under randomized
producer threads, insertion orders, worker counts, completion notifications,
cache layouts, and physical-slot histories. Canonical outcomes/bytes/hashes
must remain identical on every repeated run on the same machine. Fixtures run
at least the minimum, midpoint, and maximum placement splits with distinct
simulation-unit IDs and cell extents, prove those fields change the
configuration fingerprint, and prove mismatched replay/restore is rejected.
No cross-machine conclusion is drawn from these results.

The oracle also registers one participant RNG stream with a published toy
algorithm contract. Golden tests cover seed decoding, every state/output step,
snapshot state bytes, reconstruction from genesis/log, state-digest changes,
rollback, checkpoint, replay, exhaustion, undeclared stream rejection, and
proof that Moria itself requests no entropy.

### TECH-060 — Headless Bevy contract tests

Implements: REQ-002, REQ-005, REQ-012, REQ-015, REQ-016, REQ-023

Facade, admission, receipts, observations, lifecycle, shutdown, and failure
state machines are tested in small Bevy `App`s with `MinimalPlugins` and
deliberately selected Moria plugins. Tests seed only required resources, call
`app.update()` explicitly, and use controlled completion sinks; they never
open a window or depend on wall time.

Required slices cover every legal transition and invalid transition rejection,
permit drop/queue close, receipt drop after submission, cancellation lifetime,
full TECH-070 callable-shape compile/use, ownership return on every admission
rejection, every TECH-021 family/phase/cancellation/retry/device-loss/shutdown
row (including public replay), gap and bounded resnapshot/resume, subscription creation/start/filter/
close/drop, interest withdrawal with pins, checkpoint failure,
participant duplicate/late completion, participant source-token immutability,
private correction success/abort, snapshot export failure, completion-bridge
reservation/exhaustion/duplicate/old-generation drain, correlation propagation
and gap expiry, query minimum-revision wait/stale behavior, shutdown with dirty
truth, and a missing `RenderApp`. The missing-renderer case reports
`BackendUnavailable`; it is not a GPU test and does not install a no-op
canonical backend.

The external facade slice constructs and pattern-matches every
`AdmissionCode` with its legal `AdmissionContext`. It specifically proves
tick `BeforeNextTick`/`AfterNextTick` supplied-versus-next values,
`InvalidBatch`, `InterestTooLarge` exact brick counts, and
`ResultCapacityExceeded` exact five-field `QueryCapacity` values. Query polls
observe each `QueryReadinessReason` through
`OperationProgress.blocker` for cold ranges, materializing ranges, unmet
minimum revisions, and resource pressure; failed truth and retained-frontier
age terminate through `QueryUnavailable::Availability`, while a post-admission
output-bound proof failure terminates through
`QueryUnavailable::ResultCapacityExceeded`. No test accepts a fieldless prose
surrogate for these outcomes.
It also pattern-matches all three concrete `TelemetryError` variants, proving
unknown/closed are nonretryable and busy is retryable without a hidden
`OperationError`. Every tick-global route—canonical arithmetic/budget,
participant, base/provider, device loss, shutdown, and internal validation—is
polled as `ReceiptState::Failed(FailedNoAdvance)` and asserts the exact
attempted tick, source `FrontierPosition`, structured participant/provider/
generation identity where applicable, `committed: None`, and matching retry
policy.

A dedicated genesis-numbering fixture verifies that `GenesisReady.frontier`
is `FrontierPosition::Genesis`, `next_tick == Tick::from_raw(0)`, no
confirmed-tick rollback entry or tick outcome exists yet, and the sequence-zero
replay request is `ReplayAppendRange::Header { starting:
FrontierPosition::Genesis, next_tick: Tick::from_raw(0) }`. It then confirms
batch zero, observes
`FrontierPosition::Confirmed(Tick::from_raw(0))`,
and proves that only tick one is next. Wrong `BeforeNextTick`/`AfterNextTick`
classifications are checked on both sides of the genesis boundary. Canonical
encoding/hash fixtures distinguish `Genesis` from `Confirmed(Tick(0))`. The
fixture also queries collision truth at Genesis and after confirmed tick zero,
asserting byte-distinct `CollisionFact.source_frontier` and TECH-053 artifact
headers. Its CPU and GPU participants inspect genesis token metadata and the
224-byte `moria-participant-io-v1` position fields, then inspect tick-zero
source/destination metadata; no case may encode Genesis as confirmed zero or
as an unlabeled zero tick.

Observation fixtures move and retire a volume, reclaim the old directory
version, then poll an older retained record through volume/kind/spatial
filters. Matching must use the stored append-time bounds and membership.
Count- and byte-triggered gaps each pin a bounded resnapshot, allow new events
to arrive while it runs, resume at the captured cursor, and produce either the
exact suffix or another honest gap.

Base-content fixtures prove that all 2,048 payload bytes and callback permits
are reserved before source invocation; exercise exact/short/long writes,
invalid cells/digest, bounded diagnostics, drop, cancellation during write,
duplicate/late/generation completion, source panic, and resource release.
They assert that no source-owned `Vec`, writer, error chain, or diagnostic
allocation enters a Moria queue.

Provider-registry fixtures call every builder registration method, reject
duplicate IDs without replacement, and fail freeze for every missing or
wrong-kind base-authority/source/content-store/input-source/checkpoint-store/
replay-sink reference. Store fixtures compile external implementations of
`ContentBlobStore`, `CheckpointStore`, and `ReplaySink`; exercise exact,
known-length short, long, dropped, duplicate, cancelled, and late-generation
sink methods. Unknown-length manifest loads accept actual lengths from zero
through the maximum only at the sink layer, then reject empty, truncated,
bad-checksum, declared-length-mismatched, and trailing-byte manifests during
framing; manifest-referenced blob loads reject any cursor unequal to
`expected_bytes`. The fixture proves `load_manifest(key)` sees either the
complete atomically committed manifest or `NotFound`, never a partial value.
Checkpoint, restore, shutdown checkpoint, and recovery each use the exact
request store ID and never fall through after that store fails.

The external crate constructs, fills, reads, iterates, and consumes every
bounded owner through the TECH-070 methods, including a complete
`ReplayRequest` and the `OwnedBytes` received by each store. It asserts
unchanged values on failed construction/push, no partial byte extension,
UTF-8/64-byte validation, exact length/capacity, and admission charging.
It also constructs and round-trips every private-field ID, digest, key, and
lineage through its normative constructor/accessor family. It rejects zero for
every nonzero scalar ID, rejects `0x8000_0000` for each high-bit-reserved ID,
rejects the all-zero `ReplayStreamKey`, and proves no public tuple constructor
or unchecked conversion compiles; unconstrained fixed-byte and counter types
preserve zero and every input bit. `RngStreamId` is named separately:
`try_from_raw(0)` fails, while `0x7fff_ffff`, `0x8000_0000`, and
`u32::MAX` all construct and round-trip because its participant-local domain
is the complete nonzero `u32` range.
It constructs `PlacementScalar`, `TurnQ32`, `SimulationUnitId`,
`PlacementFixedFormat`, and `WorldGenesisConfig` only through their exact
integer/byte APIs. Compile-fail fixtures reject implicit or `From` conversions
between placement values, local cell coordinates, density, orientation,
participant representations, and `f32`/`f64`. Every participant descriptor
Format fixtures accept splits 0 and 16 and cell extents 1 and `i32::MAX`, and
reject split 17 plus cell extents 0 and 2,147,483,648 before `begin_world`.
Every participant descriptor
registers a sorted/unique bounded representation-contract list; duplicate,
zero, oversize, or changed digests fail before genesis, while an empty list is
valid for a participant with no non-placement physical quantities.
Its CPU participant downcasts its own state lease, iterates exact replay
record views, decodes a TECH-053 collider view, and exercises wrong-type,
capacity, cancellation, and dropped-lease cases. Its GPU participant creates
pipelines from `io_bind_group_layout`, binds every primary wrapper, inspects
all ranges/capacities/metadata—including every operation row's optional
source/destination frontier and attempted tick—writes
status/effect/event/state/snapshot
records, and proves mixed attempts and bad ranges are rejected at Moria
wrapper construction, while an incompatible pipeline selected before or after
infallible `bind_io` is captured by the balanced post-encoding wgpu validation
scope. Both that scoped error and overflow, missing status, or stale generation
fail before publication.

Genesis configuration supplies a nonzero consumer-chosen replay stream key.
Two worlds using the same `(ReplaySinkId, ReplayStreamKey)` in one client
reject the second freeze without a sink call, while distinct pairs reach their
exact sequence-zero request. A post-genesis append-failure fixture confirms
tick `t`, then fails that append: the tick receipt remains `Ready`, the world
enters `Failed`, the exact record stays pinned, new ticks are rejected, one
world-lifecycle observation carries the bounded `ReplayExportFailure`, the
actual `FailureCounter { code: ErrorCode::StoreFailure, ... }` and replay
failure/high-water telemetry appear, and `ShutdownReport` repeats the same
sink/stream/sequence/append-range/length/digest/failure metadata before
releasing the raw bytes. Device loss
does not rewrite the request; wrong/late completions cannot recover the world.
With otherwise default budgets, an accepted genesis failure after its
sequence-zero call consumes one retired-stream slot but releases the sole world
slot; a new builder with a distinct stream reaches genesis, while the retired
pair still returns `DuplicateId`. Four such invoked streams consume the
default retired pool and the fifth returns
`RetiredReplayStreamCapacity` before a sink call.

A restore-continuation fixture restores a checkpoint at nonzero tick `t` into
a builder with a fresh stream. It proves no world publishes while the
checkpoint-anchored sequence-zero append is pending, verifies the header's
exact store/key/manifest digest/frontier/next-tick fields, and fails without a
world on wrong/drop/store failure. After matching durability,
`RestoreReady.replay` reports sequence zero/next one; confirming tick `t + 1`
produces sequence one after that header. Cancellation before invocation
releases the pair/tombstone, whereas cancellation after invocation drains and
retires it. Duplicate-pair and retired-pool failures return the unchanged
builder/request without a sink call.

The mechanical public-closure test parses the normative snippets and rejects
any capitalized public name without an owning row, duplicate field, stale
alias, or budget-field reference not present in the exact TECH-017 nested
schema. Its retained negative fixtures include `max_log_ticks`,
`max_log_bytes`, `recovery_replay_cap`, and `PresentationState::Failed`; the
only accepted spellings are `rollback.log_ticks`, `rollback.log_bytes`,
`rollback.recovery_replay_ticks`, and `PresentationStatus::Failed`.

The participant slice exercises every row of both
`ParticipantFailurePolicy` variants at genesis, ordinary preparation,
correction, durable restore, device loss/recovery, checkpoint export, and
shutdown; it asserts that no row can publish without the participant. It also
registers two participants and proves registration rejects same-tick
dependencies, both read only `SourceState(n)`, effects resolve solely by
`(ParticipantId, local_sequence)`, opaque event bytes reach receipts/
replay after confirmation but never the Moria-state observation ring, and no
same-tick event handoff or
behavior vocabulary exists.

One generated configuration suite varies every `ResourceBudgets` field at
zero/min/default/max/max+1 and exercises every TECH-036 cross-limit inequality,
including checked-arithmetic overflow. Passing genesis retains exactly the
declared queue/pool/output capacities in telemetry; failing genesis invokes no
consumer callback and allocates no device page.
An explicit `default-budget-smoke-v1` uses every normative `ResourceBudgets`
default, a qualifying baseline adapter with the declared 2 GiB authoritative
ceiling, the default 65,536-volume capacity, and aggregate participant
frontier claims of 64 MiB. It asserts the exact 1,988,100,096-byte
`required_20_bytes` calculation after exercising both 4,096 direct inputs and
4,096 participant placement effects per tick, reaches `GenesisReady`, and
retains 20 frontiers. A companion changes only `changed_bricks_per_tick` to
16,384 under the 2 GiB byte defaults and must reject before callback/device
allocation.

`moria-qualify` compiles as a separate binary crate and imports only public
`moria`. A lint/test fails if it enables a test-only facade feature or imports a
private module. A source/AST lint fails if canonical Rust or WGSL outside the
one-way presentation conversion contains float types/literals,
transcendental builtins, or bypasses `moria-fixed-v1`.

## Shader and real-GPU validation

### TECH-061 — Shader validation pipeline

Implements: REQ-007, REQ-021, REQ-023, REQ-036

`moria-qualify shaders validate` discovers every WGSL module referenced by the
crate, parses and validates it with the exact matching Naga version and
declared capabilities, verifies reflected bindings against ABI tables, and
runs negative fixtures at the expected layer.

Negative fixtures cover malformed/semantically invalid WGSL, mismatched
bindings, undersized effective ranges, invalid workgroup dimensions/storage,
nonuniform barriers, over-dispatch without bounds guard, scan/output/counter
overflow, invalid indirect offset/range/dimensions, unsupported optional
feature, map/decode length/alignment errors, duplicate sparse reservations,
failure after reservation, tombstone reachability, and stale generations.
Each passes only if its named layer returns its named error; crash, timeout,
unrelated error, or silent no-op fails.

Every pushed error scope is popped and its result recorded. Unexpected
uncaptured validation errors fail the run.

The command also regenerates and validates TECH-071's CPU/WGSL fixed-math
sources and CORDIC table, checks every supported fractional split, and emits
the complete TECH-035 kernel-contamination inventory. A missing entry point,
unclassified atomic, unwritten output/padding byte, order-dependent
compaction, or canonical dataflow from a physical/race/arrival identity fails
at this layer.

### TECH-062 — Real-GPU semantic parity

Implements: REQ-001, REQ-023, REQ-026, REQ-036, REQ-038

The real-GPU suite exercises public genesis, queues, participant adapter,
canonical transition, collision, readback, checkpoint, and replay. For each
fixture it initializes every byte, dispatches through the normal Bevy device,
copies ordinary output to the production staging path, maps/decodes, and
compares exact bytes and integer values with the CPU oracle.

Required sizes include empty, one item, partial tile, exact tile, multi-tile,
multi-level scan, maximum command capacity, and overflow. Sparse cases include
empty/near-full/full resident directories, collision-heavy keys, 32-probe
exhaustion, duplicate contenders, tombstone-heavy caches, allocation-counter
edges, failed construction, multi-brick atomic failure, old-root readers during
publication, delayed reclaim, and compaction preserving every live mapping.
Bridge fixtures hold a decoded canonical completion in the render world and
prove the main world still exposes the old bundle, then drain the reserved
envelope and prove root, receipt, replay, participant, and observation fields
change in the same exclusive publication system.

The suite intentionally destroys/loses a test device where the backend harness
supports it. Outstanding work must fail, old callbacks must not publish,
device-bound state must reconstruct or reach the documented terminal failure,
and root semantics after successful replay must match.
It also compares exact rotated dynamic-volume collision fact wire bytes,
including world-space contact points and directed normals, rather than only
hit membership.

For replay evidence the suite executes the identical genesis and retained
`TickBatch` stream at least eight times on the same physical machine. Between
runs it perturbs worker counts, submission chunking, staging contents,
resident-slot histories, workgroup scheduling pressure, callback order, and
compaction input layout. It compares every canonical record and hierarchical
hash byte-for-byte. The kernel-contamination audit remains required even when
all sampled runs pass; sampling does not excuse an order-dependent atomic,
race-produced byte, or arrival-order compaction. Adapter/driver/backend
identity is recorded only as run context, and no cross-machine comparison or
qualification status is produced.

Evidence records input/output digests and decoded comparisons. “Shader
compiled,” “queue submitted,” “frame rendered,” a mock, or a software adapter
cannot satisfy this contract.

## Scenario acceptance

### TECH-064 — Public-boundary and truth scenarios

Implements: REQ-002, REQ-003, REQ-010, REQ-011, REQ-013, REQ-019, REQ-020, REQ-023, REQ-044

The permanent scenario suite uses consumer-supplied fixtures and the public
facade:

1. **Public boundary:** configure, verify genesis, interest, sample/region/
   trace/overlap/sweep, mutate, create filtered observation subscriptions,
   force/recover a gap, read telemetry, checkpoint, correct, restore, request
   a private-world public replay from exported records, produce a bounded
   earliest-divergence artifact from one poisoned expected hash, request device
   recovery where supported, and shut down through every TECH-070
   callable without a private import.
2. **Deep volume:** a static volume has voids, signed-density boundaries,
   material bands, and authored structures across all axes. Queries and edits
   operate at the deepest bounds; no height field exists in the fixture.
3. **Dynamic volume:** query and collide, rotate/translate by an admitted
   placement input, edit local matter, checkpoint, and restore the same ID.
4. **Atomic mutation:** inject a post-admission diagnostic failure into a
   multi-brick command before publication by constructing a public
   `ExecutionPolicy::Candidate` with TECH-040's one-shot
   `AfterBrickConstructionBeforePublication` diagnostic. Every targeted old
   cell/revision/observation remains unchanged. `moria-qualify` imports that
   public type exactly as an external crate; the failure is the same checked
   production diagnostic record and cleanup route, not a storage or mutation
   hook.
5. **Truth versus view:** remove presentation, make it stale, corrupt/discard
   derived buffers, and rebuild. Matter/collision/hash bytes remain identical;
   smooth and crisp surfaces show honest exposed cuts when current.
6. **Assembly/dressing:** an ordinary small volume supplies material-backed
   occupancy while dressing loses support and disappears without collision or
   persistence residue.

Visual captures are diagnostic evidence for smooth/crisp presentation and
dressing support. Humans review them, but exact pixels are not canonical and
cannot replace state/readback assertions.

### TECH-065 — Lifecycle, failure, and isolation scenarios

Implements: REQ-009, REQ-012, REQ-016, REQ-018, REQ-021, REQ-022, REQ-040

One machine-readable failure matrix injects every product-design condition:
cold query, invalid bounds, stale precondition, source unavailable/invalid,
closed tick, arithmetic/range exhaustion, canonical/query/pool pressure,
presentation failure, observation overwrite, store failure, corrupt/incomplete
checkpoint, wrong lineage/root, missing material/source, activation mismatch,
participant restore/divergence, rollback outside window, replay poison,
unsupported device capability, injected determinism violation, and external
participant failure.

Each row states expected layer/code, retryability, receipt terminal state,
committed tick/revision change, observation, and retained dirty state. The test
asserts unknown never becomes empty, no scar disappears, no internal storage
handle escapes, and no timing-selected tick confirms.

External participant rows are expanded for every
`ParticipantFailurePolicy`/failure-site pair in TECH-029, including
`RecoveringParticipant`, successful explicit recovery, recovery mismatch, and
terminal-world behavior. Additional rows cover participant effect overlap and
precondition conflict, event count/byte/record overflow, duplicate event
sequence, wrong schema, and proof that no output reaches a consumer when the
tick does not confirm.

I/O, materialization, callback, and presentation schedules are permuted while
the same activation input bytes run. Simulation-domain normalized bytes and
hashes must match. Overlapping activity regions produce one union. Camera and
presentation interest changes alone produce no canonical byte change.

### TECH-066 — Persistence, snapshot, and participant scenarios

Implements: REQ-014, REQ-029, REQ-030, REQ-032, REQ-035, REQ-037, REQ-038

Persistence tests checkpoint edited static and edited/moved dynamic volumes
while a later tick commits. The checkpoint reports only its pinned frontier;
the later root remains dirty. Saved data contains scars/continuation but no
mesh and is materially smaller than a raw dump of the fixture's untouched
domain.

Restore compares canonical logical state, cell queries, placements, IDs,
allocator, simulation domain, participant commitments, and root hash. It then
rebuilds GPU/cache/presentation state and compares observable behavior. Wrong
lineage with the right label but wrong exact root must fail.

Rollback retains 32 ticks (therefore at least 20) while overlapping/disjoint
bricks, placements, IDs, participant RNG commitment bytes, and domain
membership change. It restores ticks 1, 5, 10, 20, and 32 without enumerating
untouched matter, then replays to identical per-tick outcomes/hashes. One
snapshot participant and one reconstructible participant exercise success,
capacity exhaustion, missing state, and divergence. Pin/reclaim counters prove
state is live exactly while reachable.

The snapshot participant exports distinct recognizable bytes at every
frontier. Tests hold participant blob puts pending while scar puts complete and
prove the manifest cannot commit or report durability; wrong bytes/digest,
oversize, export cancellation, store failure, and device loss all leave no
manifest. Restore receives the exact stored bytes and returns a staged token.
Correction tests make the snapshot participant succeed and the reconstructible
participant fail after several private ticks, then prove the original live
participant tokens/commitments and substrate root are unchanged and every
staged token is reclaimed after its last GPU use.

A correction-branch fixture confirms an original suffix whose inputs and
hashes differ from a replacement suffix, then holds the one
`CorrectionBranch` append pending. It proves the old frontier, rollback deque,
active log, physical stream position, observations, and presentation dirty set
remain installed until matching durability. Cancellation before invocation
aborts; cancellation afterward is `NotCancellable`. Sink failure leaves those
old values installed, reports the exact branch `ReplayExportFailure`, and
terminally closes the world. The external-style consumer pattern-matches
`CorrectionReceipt::poll` as
`Failed(CorrectionError { original_frontier, error: OperationError {
code: StoreFailure, committed: None, .. },
replay_export_failure: Some(...) })`, the paired
`CorrectionObservation { from: original_frontier, to: None, replay: None,
failure: Some(StoreFailure) }`, and the paired
`WorldLifecycleFact { state: Failed, frontier: original_frontier,
failure: Some(OperationError { committed: None, .. }),
replay_export_failure: Some(...) }`. All three frontier bytes equal a public
query of the last readable frontier. The same fixture separately fails an
ordinary tick append and proves its lifecycle error instead carries
`CommittedEffect::Frontier(the_confirmed_tick_frontier)` while the already
`Ready(TickConfirmed)` receipt remains unchanged. Matching branch durability
performs one publication: the active in-memory suffix and retained-frontier
suffix contain only corrected records/roots, the physical prefix digest
advances by exactly one sequence, and the active-history digest drops the
superseded records. Existing readers of the old suffix remain valid until
their pins drain. A companion holds the last ordinary tick append pending and
proves correction admission returns `PersistenceBackpressure` before
participant or branch work.

Correction expected-hash fixtures construct a four-batch contiguous
replacement. An empty `expected_hashes` vector is admitted and performs no
consumer-supplied root comparison. Four hashes map in vector order to
`target + 1` through `target + 4`; exact values succeed, and poisoning any one
fails before private advance past that tick with `ReplayDivergence`,
`CommittedEffect::None`, no branch sink invocation, and the original frontier
still live. Nonempty vectors of lengths one, three, and five are rejected
before pins, participant callbacks, or sink work as
`CorrectionHashCountMismatch` with exact
`CorrectionExpectedHashCount { replacement_batches: 4, expected_hashes }`.
Each rejection returns the complete `CorrectionRequest`, including all four
sealed batches and every supplied hash, byte-for-byte unchanged.

The reconstructible participant checkpoint persists several
`moria-checkpoint-replay-v1` chunks, terminates the process fixture, discards
all in-memory log state, and restores using only manifest-referenced
digest-verified blobs. Its declared RNG stream must reproduce every
intermediate and final state commitment. Missing, reordered, overlapping,
gapped, corrupt, oversized, and store-failed chunks prevent manifest commit or
restore as appropriate. Holding replay blob puts pending while every scar and
snapshot blob is durable proves that the manifest cannot commit and the
checkpoint is not a recovery anchor; byte accounting includes those chunks.

Public replay tests retain/export exact live records through a registered
`ReplaySink`, discard every live world/root/log object, then pass only the
owned `ReplayRequest` header and records to a new frozen builder. Every tick's
expected root/outcome/participant bytes are compared before private advance;
success still withholds the world while it copies the exact verified header
and records into the builder's fresh stream as sequences zero through `N`.
`ReplayCompleted.replay` reports that durable prefix and the first newly
confirmed tick appends as `N + 1`. A checkpoint-anchor replay first restores
the exact header-named store/key/manifest using `anchor_restore`; missing or
extraneous restore limits reject. Bootstrap sink drop/failure, cancellation,
or wrong completion publishes no world and retires the pair only after its
first invocation. Poisoning each expected category
returns the earliest exact `DivergenceArtifact` without publishing a world.
Fixtures cover record/sink count and byte backpressure, dropped/duplicate/
late sink completion, absent/gapped/reordered records, cancellation before and
after private submission, artifact-capacity rejection, and device loss.
Replay-identity fixtures run the same frozen world with two distinct
`ExecutionSummary.adapter_context` values and prove the sequence-zero header,
`ReplayIdentityV1` digest, record digest, and durable-prefix digest are
byte-identical; restore and public replay reach their first transition without
`ContractMismatch`. Changing only the placement split, cell extent,
simulation-unit ID, TECH-071 arithmetic/table digest, or another
configuration-fingerprint input changes the identity digest and returns
`ContractMismatch` before a participant callback, device submission, canonical
transition, or destination-sink invocation. An invalid authority-status tag
and a status mismatch fail at that same pre-transition boundary.
Golden cases independently recompute
`ReplayStreamPosition.durable_prefix_digest` from each ordered
`(sequence, record_digest)` tuple after the header, after every copied record,
and after the first new tick. `ReplayCompleted` exposes no second ambiguous
sequence digest.

The cold-replay fixture includes at least one correction branch. It supplies
the complete physical stream, proves the superseded suffix is processed only
as branch-verification evidence, and reproduces only the corrected outcomes,
participant/RNG commitments, events, roots, and final hash. Removing,
reordering, or poisoning the branch or either active-history digest fails at
the branch sequence. A checkpoint taken after the same correction stores only
the active corrected tick frames (including frames extracted from the branch
record); after process teardown, checkpoint restore reconstructs both
participant strategies and the corrected live root. Supplying the superseded
pre-branch frames instead must fail commitment/root verification.

Poisoning each canonical influence category changes its appropriate leaf/root
at the first affected tick. Poisoning mesh/cache/telemetry does not.

## Performance and evidence

### TECH-067 — Fixed rollback performance fixture

Implements: REQ-007, REQ-022, REQ-041

Reference fixture `rollback-chain-v1` is frozen before measurements:

- 8 static sparse volumes with a combined 65,536 resident 8³ bricks;
- 64 dynamic volumes, each 8 bricks, with independent placements;
- 2 external participants: one GPU `PerTickSnapshot` participant with a
  2 MiB maximum snapshot and one
  `ReconstructibleFromCanonicalStateAndLog` participant with 32-tick capacity;
- a consumer-defined 64-link interaction dependency chain represented only in
  participant input/state (Moria contains no physics/constraint meaning);
- 256 changed bricks and at most 8,192 changed cells per tick, half overlapping
  prior-tick dirty regions;
- activation/deactivation of 32 brick regions and 16 placement changes per
  tick;
- rollback depths 1, 5, 10, 20, and 32;
- simulation-frame interval 16.667 ms.

After 30 warm-up corrections, each depth runs 120 corrections. Evidence reports
median, p95, maximum, CPU wall time, GPU timestamps when supported, mapping
cost, participant time, restored/replayed ticks, bytes retained/changed/hashed,
and complete restore-through-final-replay time. The **20-tick performance
tier** requires p95 complete correction at depth 20 no greater than 16.667 ms
on the named tuple. Correct slower tuples report their measured rollback-per-
16.667-ms curve and do not claim the tier.

GPU timestamps are measurement only, never synchronization. Runs without them
retain CPU wall-time evidence and mark GPU timing unavailable. Correctness must
pass before performance supports a claim.

### TECH-068 — Quality benchmark receipts

Implements: REQ-007, REQ-022, REQ-023, REQ-041

The approved design's performance policy supersedes the pre-amendment
architecture's `P1`–`P10` feasibility gates. Those identifiers are not v1
acceptance gates, are not silently inherited thresholds, and are not claims
made by this TDD. V1 has:

1. universal correctness, boundedness, determinism, and fail-closed gates from
   TECH-059 through TECH-066 and TECH-069;
2. the single named 20-tick rollback performance tier in TECH-067; and
3. hardware-contextual benchmark receipts below, whose measured distributions
   are reported without converting old thresholds into pass criteria.

`moria-qualify benchmark` fixes fixture digest, budgets, warm-up, sample count,
interest route, mutation density, and adapter context before running. Separate
receipts report:

- sparse logical/resident brick ratio, homogeneous-region compression,
  interest churn, eviction/reclaim, and authoritative/derived residency
  high-water marks;
- base callback, scar-overlay, upload, and ready-transition materialization
  time/bytes, including cold and cache-hit paths;
- mutation admission-to-confirm, changed-brick/hash work, and
  confirm-to-current-presentation;
- point/region/trace/overlap/sweep query and canonical-collision response,
  inspected records, result/readback bytes, and oracle parity;
- presentation queue/build/upload/install time, vertex/index bytes,
  revision lag, stale/coalesced work, and failure isolation;
- lifecycle transition latency and queue/pool high-water marks;
- authoritative, rollback, scratch, readback, and derived GPU bytes;
- checkpoint pin/readback/blob/manifest throughput and size, plus cold
  restart restore/rebuild time;
- CPU/GPU participant input/artifact/effect/event/state/snapshot handoff,
  transfer/readback pressure, commitment cost, and generation recovery;
- incremental leaves/nodes/bytes hashed;
- replay ticks/second, retained bytes, root-install cost, and complete
  correction-depth curves including depths 1/5/10/20/32.

Every receipt names CPU/GPU/driver/backend, granted limits/features, budgets,
fixture density/scale, warm-up/sample count, timestamp availability, mapping
cost, and software versions. No universal latency, throughput, residency
ratio, or memory target is claimed beyond TECH-067's named tier. Correctness
rows have `PASS`, `FAIL`, or `NOT_DEMONSTRATED`; budget exhaustion is `FAIL`.
Performance rows report measured values and `TIER_MET` only for the named
rollback tier, otherwise `MEASURED` or `NOT_DEMONSTRATED`. Reports from
differing machines remain comparable because raw configuration/context and
distributions are retained, not only a summary score.

### TECH-069 — Evidence schema and completion gate

Implements: REQ-021, REQ-022, REQ-023, REQ-026, REQ-044

Evidence lives under a caller-selected directory and uses
`moria-evidence-v1` JSON manifests referencing immutable binary blobs by
BLAKE3. A run manifest includes command line, fixture/contract/source/commit
digests, dirty-worktree status, environment, adapter, limits, results,
measurements, expected/actual errors, artifact paths, and missing claims.
Canonical byte blobs use their native versioned formats rather than JSON
numbers.

The implementation-ready completion gate is:

1. mechanical traceability proves every current approved `REQ` is implemented
   by at least one semantically authorized stable `TECH`, with REQ-039 retained
   only as the explicitly superseded cross-machine record; every active
   `TECH` has exactly one matching `Implements:` line, and `traceability.md`
   has exact pair parity;
2. implementation-completeness validation proves every normative public Rust
   name is defined, every TECH-070 callable compiles from the external-style
   binary, every async family has the TECH-021 admission/cancel/terminal/
   retry/shutdown/generation tests, and every provider registry,
   key-based manifest load, resource/callback/observation/participant/replay
   ownership bound has executable evidence;
3. the exact local commands in TECH-004 pass;
4. all CPU/headless/public-boundary/failure/persistence/rollback tests pass;
5. every WGSL module and negative fixture passes at its named layer;
6. real-GPU parity and eight-run replay comparison pass on the current machine
   used to make a replay-grade claim;
7. the complete TECH-035 kernel-contamination audit and TECH-071
   arbitrary-precision CPU/WGSL differential suite pass;
8. device-loss behavior is evidenced as reconstructed or terminal;
9. visual fixtures are captured and human-reviewed only for presentation
   claims;
10. benchmark receipts report every TECH-068 path, hardware context, budgets,
    and honest status without claiming superseded `P1`–`P10` gates;
11. no missing row is interpreted as pass.

Release automation must not emit a replay-grade evidence manifest if the
worktree is dirty, contract/source digests differ, an unexpected GPU error
occurred, replay evidence is unavailable, the contamination audit is
incomplete, or same-machine byte comparison diverges. The retained artifact is
a replay evidence manifest, not an adapter/vendor/driver qualification record,
and release automation performs no cross-machine comparison.

An approval statement must name both conclusions separately: **approved GDD
coverage** means the traceability condition in item 1, while **approved
implementation completeness** means items 2 through 11 leave no undefined
callable, owner, bound, lifecycle, or evidence obligation. Passing one does not
assert the other.
