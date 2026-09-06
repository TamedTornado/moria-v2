# Bevy and GPU runtime

The GPU is the normal owner of detailed canonical matter. This document
defines how that ownership is implemented without creating a privileged
consumer path or confusing submission with completion.

## Bevy ownership and schedules

### TECH-031 — Single Bevy renderer device

Implements: REQ-005, REQ-007, REQ-018, REQ-024

`MoriaPlugin` installs focused `CanonicalPlugin`, `QueryPlugin`,
`PersistencePlugin`, `ParticipantPlugin`, `PresentationPlugin`, and
`TelemetryPlugin` implementations. The main-world side owns public queues,
receipts, content tasks, and root metadata. Device-bound buffers, bind groups,
layouts, pipelines, pools, submissions, and mapping state live only in
`RenderApp`.

The adapter obtains Bevy's `RenderDevice` and `RenderQueue`; it never requests
a second wgpu device. `ExtractSchedule` copies only bounded request descriptors,
immutable canonical bytes, and root-generation deltas. Large content moves
through owned staging permits, not cloned ECS structures.

Plugin finish creates one `Arc<RenderCompletionBridge>` and inserts a clone
into both worlds. It is a mutex-protected, preallocated fixed ring, not an ECS
event channel: extraction remains one-way, while render completions return
through this explicitly shared transport. Each admitted GPU job must reserve
one ring cell before extraction and carries
`(JobId, WorldId, DeviceGeneration, attempt_nonce)`. The compiled maximum is 32
job cells, at least the sum of the configured canonical, query, checkpoint,
materialization, genesis, and presentation in-flight slots, plus two dedicated
generation/shutdown control cells. Materialization shares the query/readback
slot pool. Exhausted cells apply admission backpressure; the
render thread never drops or overwrites a completion.

GPU root and participant tokens carried by an envelope are internal
generation-tagged IDs into render-world ownership tables, not wgpu handles.
The main-world bundle may copy those IDs for pinning and later extraction, but
only `RenderApp` resolves them to device objects; they never enter public IDs,
canonical bytes, replay, or persistence.

Device-independent plugin and ABI descriptors initialize in the main world.
All recovery-sensitive GPU resources initialize through Bevy 0.19
`RenderStartup`, so a new render device builds a new device generation.

### TECH-032 — Render schedule ordering

Implements: REQ-005, REQ-013, REQ-033, REQ-040

Moria defines render-system sets with explicit edges:

```text
Main First:
  MoriaCollectRenderCompletions
    -> MoriaPublishCanonical
    -> MoriaFinalizeOtherReceipts

Main PostUpdate:
  MoriaCoordinateRequests

ExtractSchedule:
  ExtractMoriaRequests

Render schedule / root RenderGraph:
  MoriaPrepareResources
    -> MoriaEncodeCanonical
    -> MoriaSubmitCanonical
    -> MoriaDriveCompletion
    -> MoriaPrepareDerived
    -> MoriaEncodePresentation
```

Canonical camera-independent compute dispatches run in an explicitly ordered
root `RenderGraph` subgraph after preparation and before presentation consumes
new roots. Query dispatches against already committed roots may run alongside
derived work but never interleave with candidate publication. CPU callback and
mapping progress is polled in `MoriaDriveCompletion`.

`MoriaDriveCompletion` moves a terminal envelope into its already reserved
bridge cell only after mapping and decode. On the next main-world `First`,
`MoriaCollectRenderCompletions` drains at most the configured 32 cells into a
fixed `JobId` table. The exclusive `MoriaPublishCanonical` system accepts
exactly one envelope for the sole pending canonical attempt, revalidates world,
attempt nonce, source frontier, device generation, mapped diagnostic counts,
participant products, and root hash, then performs TECH-013 step 12 as one
`Arc<FrontierBundle>` swap. That same critical section updates receipt state,
rollback deque, the active semantic replay-log projection and durable stream
position, participant commitments/tokens, revision metadata, and canonical
observations before any later main-world system can read them. For correction,
TECH-048 supplies a previously durable branch completion in the candidate
envelope; the critical section splices the rollback/log suffix and publishes
the corrected bundle together. It never exposes a corrected bundle with the
superseded semantic log.
Noncanonical job completions are finalized only afterward.

Duplicate envelopes are an invariant failure and terminally fail the
generation; an unknown or already-aborted `JobId` is acknowledged only for
lifetime cleanup. An old-generation envelope may release its reserved cell and
render resources but cannot publish. The main world never waits while holding
the bridge mutex, and callback timing cannot select a tick outcome.

Presentation may lag by frames. Extraction never waits for it. Rollback
suppresses intermediate replay-derived work and emits one final dirty-root
delta. No system set, graph node, callback, or frame boundary is a canonical
tick boundary; the coordinator's confirmed publication is.

## Physical representation

### TECH-033 — Buffer and page layout

Implements: REQ-004, REQ-010, REQ-018, REQ-033

Durable device buffers are paged so one binding never exceeds the granted
`max_storage_buffer_binding_size`:

| Pool | Record | Baseline page |
| --- | --- | ---: |
| Dense brick | 2,048-byte `CellWire[512]` | 32 MiB / 16,384 bricks |
| Uniform brick | 16-byte aligned record | 8 MiB |
| Radix node | 1,024-byte record with 16 child handles/hashes | 32 MiB / 32,768 nodes |
| Volume metadata | 256-byte record | 8 MiB |
| Resident base directory | 32-byte bucket | 16 MiB |
| Inputs/outcomes | versioned wire records | 8 MiB per in-flight slot |
| Scan/sort scratch | phase-specific | 32 MiB per in-flight slot |
| Readback | `MAP_READ | COPY_DST` only | 16 MiB per slot |

Page sizes are ceilings; initialization chooses the lower of configured budget
and actual device limits. Storage binding offsets are multiples of
`min_storage_buffer_offset_alignment` and each effective range covers only its
page. A separate page table resolves logical handles; a shader never assumes
one monolithic allocation.

Canonical work buffers use `STORAGE | COPY_SRC` (plus `COPY_DST` where
initialization requires). Readback always copies into a distinct
`MAP_READ | COPY_DST` buffer. No buffer is mapped while submitted for GPU use.

The resident base directory is a bounded open-addressed cache with explicit
`EMPTY`, `RESERVED`, `OCCUPIED`, and `TOMBSTONE` states and maximum 32 probes.
Reservation, payload write, duplicate validation, and publication are separate
ordered dispatches. This directory is noncanonical: a miss changes readiness,
never truth. Canonical scars use the immutable radix tree rather than a mutable
hash table.

### TECH-034 — Host/WGSL ABI

Implements: REQ-007, REQ-021, REQ-028, REQ-036

Every wire type has:

- an explicit Rust encoder/decoder rather than transmute;
- a WGSL struct using only `u32`/`i32` scalar words;
- a table of field offsets, alignment, size, array stride, and binding range;
- compile-time Rust size/offset assertions where a staging mirror exists;
- a shader fixture that writes each field and padding word for exact readback;
- zero initialization of padding, counters, flags, and unused output slots.

Rust `bool`, enums, pointers, `usize`, `isize`, and implicit struct padding are
forbidden in GPU ABI. `CellWire` pairs are packed/unpacked through `u32`.
`vec3` is not used in durable records. Runtime arrays appear only as the last
member and are bound to an exact logical record count; `arrayLength` is not
used to infer allocation capacity.

Dispatch count is computed with checked `count / width + (count % width != 0)`.
Every over-dispatched invocation checks logical count. Workgroup dimensions,
invocation product, workgroup memory, buffer range, and indirect `[x,y,z]`
records are validated against granted limits before encoding.

## Portable canonical kernels

### TECH-035 — Kernel synchronization and deterministic output

Implements: REQ-007, REQ-026, REQ-028, REQ-033, REQ-036

Canonical WGSL uses the portable 32-bit baseline. Atomics may claim
noncanonical work slots or set failure flags; no atomic winner supplies stable
identity, output order, revision, or hash input. Multiword records follow:

```text
reserve -> ordered payload initialization -> validate -> ordered publish
```

Cross-workgroup phases use separate dispatches on the same queue. Barriers are
in uniform control flow and all inactive lanes contribute identity values.
There is no spin wait, “last workgroup,” subgroup-width assumption, 64-bit CAS,
or device-scope publication assumption.

Variable canonical output uses fixed slots or stable
mark/scan/scatter. Atomic append is permitted only for bounded noncanonical
diagnostics and reports overflow. The portable scan hierarchy and radix sort
are specified in TECH-015. Indirect dispatch records are exactly three packed
`u32` values, four-byte aligned, range checked, and generated in a distinct
phase; if `INDIRECT` is unsupported the equivalent host-known bounded direct
dispatch is used without changing canonical results.

Each shader operation is wrapped in balanced validation error scopes. Shader
parse, Naga validation, module/layout creation, pipeline creation, encoding,
submission, mapping, decoding, and semantic comparison are distinct error
layers.

The kernel-contamination audit is a mandatory source and execution gate for
every canonical WGSL entry point and every transitive helper. The audit
records all atomic operations, workgroup/shared storage, output-slot
assignment, compaction/sort paths, padding writes, allocator/free-list reads,
hash inputs, and host callback inputs. It rejects any canonical byte, identity,
revision, allocation decision, ordering key, outcome, or hash input derived
from an atomic winner, race-produced or uninitialized byte, arrival order,
physical-slot order, subgroup/lane identity, unordered iteration, or
order-dependent compaction. This audit is required for same-machine,
run-to-run replay; cross-vendor comparison is not its rationale or acceptance
criterion. A finding blocks replay-grade execution until the kernel uses fixed
slots or stable mark/scan/scatter and the same-machine perturbation fixture
passes.

### TECH-036 — Bounded resource policy

Implements: REQ-004, REQ-005, REQ-018, REQ-021, REQ-022

`ResourceBudgets` is immutable after genesis and contains byte/count ceilings.
The complete normative field schema, defaults, and portable compiled maxima are
in TECH-017; there are no hidden queues or opportunistically growing pools.
The fields bound these retained resources:

| Group | What it bounds | Overload behavior |
| --- | --- | --- |
| `identity` | world/material/volume/participant/input/base-authority/content-store/checkpoint-store/replay-sink/RNG identities; the separate client-lifetime active/retired replay-stream reservation pool; live operation, terminal-receipt, root-pin, and artifact-lease records and retained result bytes | reject the registration/admission; an exhausted retired-stream pool returns `RetiredReplayStreamCapacity` before sequence-zero callback; held receipt/root/artifact leases apply backpressure rather than grow |
| `canonical` | sole pending tick, input/correlation bytes, command target size, changed bricks, and all candidate scan/sort/output scratch | return owned batch before admission, command failure, or `FailedNoAdvance { cause: Canonical(LogicalCapacity), error: OperationError { code: CanonicalBudget, ... }, ... }` as appropriate |
| `content` | base request queue, invoked callback sinks and bytes, one materialization job, resident dense/uniform/radix/directory records, and authoritative GPU pages | delay/reject materialization or retire eligible cache; never evict a pin/scar/frontier |
| `query` | queued/in-flight queries, inspected bricks, revision list, result records/bytes, staging slots, and aggregate readback bytes | reject and return request, wait only when request selected `Wait`, or finish capacity failure; never truncate `Complete` |
| `observation` | shared ring records plus payload bytes, subscription records and finite volume lists, poll output, and resnapshot summaries/query payload | reject subscription/resnapshot or emit explicit ring gap |
| `presentation` | dirty queue, resident chunks/bytes, in-flight jobs, vertex/index/output bytes, and dressing records | coalesce, retire, keep stale, reject, or fail the derived chunk |
| `checkpoint` | queued/active checkpoints, map/store staging slots and bytes, blob size, manifest node/blob/count bytes, and total checkpoint output | reject/queue, cancel before submission, or fail without a manifest |
| `rollback` | retained roots/bytes, explicit genesis/frontier metadata reserves, in-memory log, replay-sink in-flight records/bytes, public private-world replay and divergence artifacts, one correction, private replay ticks/bytes, and recovery replay depth | reject genesis, tick, replay, correction, or recovery; never evict a reachable or required-20 frontier |
| `participant` | concurrent callback/GPU operations, input, fixed effect/event sinks, state/snapshot tokens, collider artifacts, and checkpoint snapshot bytes | reject descriptor/genesis or cause `FailedNoAdvance`/operation failure; never resize a callback output |
| `runtime` | interest control records, all consumer callback completion cells/bytes, and render-to-main bridge cells | reject before callback/extraction or apply the owning operation's explicit backpressure |

Before `VerifyingGenesis` can invoke consumer code, checked `u128` arithmetic
validates all of the following and returns a field-path `ConfigError` on the
first violation:

1. Every field is nonzero where work is enabled, lies within TECH-017's
   min/max, and all fixed values are exact:
   `canonical.pending_ticks == 1`, `checkpoint.active_requests == 1`,
   `rollback.active_corrections == 1`,
   `checkpoint.staging_slots <= 3`, and
   `runtime.render_completion_cells == 32`.
   `identity.retired_replay_streams_per_client` is a client-level pool, not a
   per-world registry count; one permit is reserved before every sequence-zero
   replay callback and is never borrowed from `identity.worlds`.
2. `IdentityBudgets.operation_records_per_world` is at least the sum of the
   configured pending tick, base request queue, interest control queue, query
   queue, checkpoint queue, public replay attempts, active correction, one
   recovery, replay-sink callbacks, presentation dirty queue, participant
   operations, and observation subscriptions.
   Terminal receipts and their bytes are separate reserved pools; every
   accepted result-producing operation must reserve both before admission.
3. `content.base_completion_bytes_in_flight >=
   2_048 * content.base_requests_in_flight`.
   Every `BaseSourceDescriptor.max_requests_in_flight` is no greater than the
   world field; Moria may invoke fewer but never more for that source.
   `runtime.callback_completion_slots` and bytes must fit at least one largest
   registered base, CPU-participant state/effect/event/snapshot, store-load, or
   store-completion result. The scheduler may reserve fewer simultaneous
   callbacks than their queue counts, but it may never invoke consumer code
   first and seek capacity afterward. Registered content/checkpoint store and
   replay-sink descriptor maxima must be no greater than their corresponding
   blob/manifest/record and callback fields; provider counts must fit the
   identity registry fields.
4. The render bridge inequality is
   `1 canonical + query.in_flight_requests +
   checkpoint.staging_slots + presentation.in_flight_jobs + 2 control
   <= runtime.render_completion_cells`. Genesis, restore, correction, recovery,
   and GPU participant work share the one canonical cell; materialization
   shares query slots. Every extracted job reserves its cell first.
5. `query.readback_bytes_in_flight >= query.bytes_per_result`,
   every per-request `QueryLimits` is no larger than the world fields, and
   terminal receipt bytes can retain the largest admitted query or resnapshot
   result. A query result allocation reserves its full declared record and
   byte limit before encoding.
6. Observation record, poll, and resnapshot byte maxima are no greater than
   `payload_bytes_per_world`; poll/resnapshot record maxima are no greater than
   their owning count pools; and the fixed subscription volume list fits both
   identity and resnapshot summary limits. Count and byte ring limits apply
   independently; reaching either evicts whole oldest records and causes a
   gap.
7. `presentation.resident_bytes >= presentation.bytes_per_job` and each
   vertex/index count fits that job's exact encoded output. The sum of
   submitted presentation jobs may not exceed resident plus in-flight bytes.
8. `checkpoint.mapped_bytes_in_flight` fits at least one 16 MiB readback page,
   `store_bytes_in_flight >= bytes_per_blob`, manifest counts/bytes fit the
   total checkpoint limit, each replay chunk fits `bytes_per_blob`, and a
   registered participant snapshot fits both participant and checkpoint
   totals. A logical blob larger than one readback page reserves its complete
   final Moria-owned bytes first, then fills/hashes it through ordered pages;
   no mapped page or callback buffer grows. Submitted slots are additionally
   throttled by aggregate byte limits.
9. `rollback.retained_frontiers` equals
   `RollbackConfig.capacity_ticks`, is at least 20, and is no greater than
   `rollback.log_ticks`. The checked genesis worst case is the following
   conservative allocation bound, with every product and sum evaluated in
   `u128`:

   ```text
   cow_brick_bytes = changed_bricks_per_tick * (2,048 + 26 * 1,024)
   changed_volume_records =
       canonical.inputs_per_tick + participant.effects_per_tick
   changed_volume_bytes = changed_volume_records * 256
   one_frontier_bytes =
       cow_brick_bytes
       + changed_volume_bytes
       + rollback.frontier_metadata_bytes
       + participant.state_and_snapshot_bytes_per_frontier
   required_20_bytes =
       rollback.genesis_persistent_bytes + 20 * one_frontier_bytes
   ```

   `rollback.genesis_persistent_bytes` must cover the complete material,
   provider, volume, directory/root-table, allocator, and genesis participant
   records at the configured identity capacities; it is a reservation, not an
   estimate. `frontier_metadata_bytes` covers world-registry paths, input/
   outcome/root headers, and revision tables not already included in the
   per-brick/per-volume terms. The implementation may prove a lower
   registration-specific value for either explicit reserve, but it may not
   assume scar-path sharing: the formula deliberately permits every changed
   brick to lie in a different volume. `required_20_bytes` must fit both
   `rollback.retained_bytes` and `content.authoritative_gpu_bytes`.

   The volume-record term counts every direct canonical input and every
   participant effect independently because each may name a different volume;
   it does not assume overlap, no-op effects, or shared registry paths.
   With all normative defaults, the exact upper bound is
   `256 MiB + 20 * (512 * 28,672 + (4,096 + 4,096) * 256
   + 2 MiB + 64 MiB) = 1,988,100,096 bytes`, below both 2 GiB defaults. A larger
   `changed_bricks_per_tick`, participant aggregate, or registry must be
   paired with larger checked byte budgets; the 16,384 portable count maximum
   is not a viable default under 2 GiB.
   `recovery_replay_ticks <= log_ticks`; correction ticks/bytes must fit the
   log and private-root pools. Replay-sink in-flight counts/bytes, public
   replay ticks/input/private bytes, correction/replay result bytes, and
   divergence bytes must fit their dedicated fields. Each result/artifact
   maximum plus fixed receipt metadata must fit
   `identity.terminal_receipt_bytes_per_world`; concurrent admissions reserve
   against that aggregate rather than grow it.
10. For registrations sorted by `ParticipantId`, the sums of descriptor
    representation contracts, RNG streams, input, effects, events, event
    bytes, state/snapshot, and artifact claims fit the corresponding identity/
    participant fields and canonical scratch. Representation IDs are nonzero,
    sorted, and unique; every representation contract digest is bound before
    participant code runs. Every per-event maximum fits the event aggregate;
    every snapshot fits the checkpoint blob/total and callback or GPU staging
    pool. Exactly one state token record per participant per retained frontier
    is included in the rollback calculation.
11. Dense/uniform brick, radix, directory, scratch, readback, checkpoint,
    rollback, presentation, and participant pages fit checked `u64` page
    counts, the configured group byte ceilings, `max_buffer_size`, and
    `max_storage_buffer_binding_size` after TECH-033 paging/alignment. The
    actual adapter may lower a configured byte ceiling; it may not silently
    lower a logical canonical count after genesis.

One in-flight permit owns all input, scratch, output, diagnostic, and staging
resources for that job. A permit returns only after the last queue completion,
mapped views are dropped, buffers are unmapped, results are decoded/discarded,
and any staging-belt chunk has been recalled.

Canonical resource classifications use declared logical limits, not physical
allocation race or current free-list order. Callback, observation,
participant-event, query, checkpoint, and presentation outputs write only into
the Moria-owned reservation made before producer execution. If the adapter
cannot allocate the declared baseline plus 20 frontiers, genesis fails.
Noncanonical work applies priority, delay, coalescing, retirement, or rejection
as shown; it never evicts pinned truth or unsaved scars.

### TECH-037 — Completion, mapping, and generation safety

Implements: REQ-005, REQ-015, REQ-021

Internal job state distinguishes `Encoded`, `Submitted(SubmissionIndex)`,
`GpuComplete`, `MapPending`, `Mapped`, `Decoded`, and `TerminalFailure`.
`Queue::submit` is not completion. Native progress is driven through Bevy/wgpu
polling without blocking the app thread. A successful mapping callback is
required before bytes are read; views drop before `unmap`.

Every job, root, buffer handle, callback, receipt bridge, and derived artifact
carries `DeviceGeneration`. Device loss marks that generation terminal;
callbacks from it can release lifetime state but cannot publish results.
Consumer cancellation after submission discards delivery only. Old
application handles fail `StaleGeneration`.

Pool telemetry records capacity, bytes, high-water marks, oldest age,
submission-to-complete, complete-to-map, decode time, and cancellation point.
Timeouts are diagnostics only: wall-clock expiry may mark the world
environmentally failed, but cannot select or publish a canonical result.

The bridge reserves one cell for every extracted job until the main world
acknowledges its terminal envelope. Shutdown closes new reservations, asks the
render world to terminalize every extracted job, and drains all reserved cells
before removing either bridge clone. Device loss writes one dedicated
`GenerationLost` control record and terminalizes each job into its own reserved
cell. If fixed-ring accounting is violated, the world fails closed; no
completion is silently replaced.

## Device loss and recovery

### TECH-038 — Fail-closed device recovery

Implements: REQ-001, REQ-005, REQ-014, REQ-015, REQ-021

```rust
pub struct RecoveryRequest {
    pub world: WorldId,
    pub expected_frontier: FrontierSummary,
    pub store: CheckpointStoreId,
    pub checkpoint: CheckpointKey,
    pub max_replay_ticks: u32,
    pub max_replay_bytes: u64,
}
```

On device loss Moria:

1. closes admission and marks every candidate in the old generation
   `FailedNoAdvance { attempted_tick, source_frontier,
   cause: TickNoAdvanceCause::Device { generation, code:
   ErrorCode::DeviceLost }, error: OperationError { committed: None, ... } }`;
2. preserves the last trustworthy frontier identity (including
   `FrontierPosition::Genesis` if tick zero has not confirmed), tick log,
   checkpoint/genesis identity, and participant commitments on the host, but
   makes GPU queries unavailable;
3. applies every GPU participant's TECH-029 failure policy: any `FailWorld`
   participant makes the world `Failed`; otherwise the world enters
   `RecoveringParticipant` and waits for an explicit TECH-070
   `request_recovery`;
4. reserves one bounded recovery attempt, asks Bevy `RenderStartup` to
   construct a new generation, and validates the replacement adapter's
   required capabilities plus the canonical-math startup vectors;
5. restores the newest compatible durable checkpoint—including its
   participant snapshot and exact replay-record blobs—and replays the
   confirmed in-memory suffix, bounded by
   `rollback.recovery_replay_ticks` (default 256 ticks);
6. compares every retained expected hash, participant commitment, and RNG
   commitment;
7. republishes the same frontier and returns to `Ready`, or applies
   `NoAdvanceExplicitRetry` by remaining `RecoveringParticipant` with an
   explicit failed recovery receipt.

The confirmed tick log stores inputs and bounded canonical outcome/participant
records, not a voxel mirror. If there is no compatible checkpoint/genesis
source, replay exceeds its bound, a participant cannot restore, required
capabilities are absent, canonical-math vectors fail, or a hash diverges, that
recovery attempt fails explicitly.
Moria never loops or schedules a timer retry; the consumer may request another
bounded attempt or shut down. The last durable checkpoint remains usable;
Moria never fabricates empty matter or publishes a different state as
recovery.

Admission verifies that the expected frontier is still the last trustworthy
frontier, the `(store, checkpoint)` pair is a compatible visible recovery
anchor in the world's frozen checkpoint-store registry, and
request limits fit `ResourceBudgets.rollback`; rejection returns the complete
request. Acceptance reserves one private bundle, replay/log bytes, participant
state, one canonical GPU/bridge cell, and terminal receipt storage before
creating the new generation.

Device loss during checkpoint readback leaves the manifest uncommitted and the
root pinned until failure cleanup. Device-bound objects are never retained
across generations. Recovery calls that exact store's `load_manifest`; neither
the configured default nor another store is tried after failure. The
durable-frontier record and manifest must both name the same store ID and key.

## Portability and execution

### TECH-039 — Native portability baseline

Implements: REQ-005, REQ-007, REQ-024, REQ-043

Supported native adapter paths are:

- macOS physical GPU through Metal;
- Linux physical GPU through Vulkan;
- Windows physical GPU through DX12.

The baseline requires compute shaders, 32-bit integer atomics, storage buffers,
copy/map readback, at least 128 MiB effective storage binding range, at least
256 MiB buffer allocation, and sufficient workgroup limits for 128
invocations. Actual features, limits, downlevel flags, adapter identity, and
fallback status are recorded for diagnosis and benchmark context.

Subgroups, timestamp queries, native 64-bit shader integers, large binding
ranges, and indirect dispatch are optional measured branches with baseline-
equivalent public semantics. Software/fallback adapters may run diagnostics
but cannot satisfy a physical-GPU performance claim. WebGPU/WebAssembly,
WebGL/GLES, and
standalone second-device operation are excluded current targets.

This is runtime portability, not a cross-machine determinism tier. Moria makes
no claim that different machines, GPU vendors, drivers, or backend families
produce the same complete simulation hash sequence. There is no cross-GPU
qualification matrix, per-driver requalification record, cross-vendor CI gate,
Metal/Vulkan cross-vendor kernel spike, or DX12 qualification bookkeeping.
Driver/backend identity is diagnostic context only. A conformance fixture and
a later human-authorized contract would be required before adding a
cross-machine determinism claim.

### TECH-040 — Replay-grade and diagnostic execution modes

Implements: REQ-008, REQ-021, REQ-023, REQ-026

`ExecutionPolicy` and its result summary are closed:

```rust
pub enum ExecutionPolicy {
    ReplayGrade,
    Candidate { diagnostics: CandidateDiagnostics },
}

pub struct CandidateDiagnostics {
    pub fault_once: Option<CandidateFaultOnce>,
}

pub struct CandidateFaultOnce {
    pub tick: Tick,
    pub command_order: CanonicalOrder,
    pub stage: CandidateFaultStage,
}

pub enum CandidateFaultStage {
    AfterBrickConstructionBeforePublication,
}

pub struct ExecutionSummary {
    pub status: AuthorityStatus,
    pub configuration_fingerprint: ContractDigest,
    pub adapter_context: ContractDigest,
}
```

`ExecutionPolicy` has two modes:

- `ReplayGrade`: the normal authority mode. Genesis requires the declared
  device capabilities, exact canonical contract/configuration fingerprint,
  generated TECH-071 constant-table digest, and fixed canonical-math startup
  vectors to pass. It promises same-machine replay only: on the same physical
  machine, the same genesis bytes and `TickBatch` bytes produce the same
  canonical bytes and hash sequence on every run.
- `Candidate { diagnostics }`: runs the identical public API and kernels to
  produce evidence, but every frontier is labeled `DiagnosticCandidate` and
  cannot be exported as an authoritative checkpoint.

`CandidateDiagnostics` is a normal public configuration type available to any
external consumer. It is bounded to one optional
`FaultOnce { tick, command_order, stage }`; v1 has the single stage
`AfterBrickConstructionBeforePublication`. The selected ordinary matter
command passes normal admission and construction, after which the production
diagnostic/status record is set to `InjectedCandidateFailure` before
TECH-013 validation step 9. The coordinator then follows the ordinary
`FailedNoAdvance` cleanup path. The fault plan cannot write cells, roots,
outcomes, or buffers, cannot target replay-grade mode, is not a canonical
input, and is recorded in candidate evidence. Replay-grade configuration
rejects nonempty diagnostics. This seam is diagnostic control, not a mutation
bypass or a self-reported correctness result.

There is no automatic CPU, alternate GPU, or relaxed-shader fallback in a
replay-grade world. A canonical shader, encoding, math table, arithmetic,
hashing, transition, placement format, or participant contract change creates
a different configuration fingerprint and TECH-047 replay identity. A driver update
does not require bookkeeping or requalification; it must still pass the same
startup vectors and mandatory same-machine replay/contamination tests.
Presentation-only changes do not affect the fingerprint unless they change
shared bindings or scheduling relevant to canonical work.
