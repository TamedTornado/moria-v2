# Public interfaces and state machines

This document specifies the consumer-visible Rust contract. Signatures are
normative shapes; implementation may add lifetimes and private fields but may
not weaken ownership, bounds, or outcomes. The facade is runtime-neutral:
progress is driven by Bevy schedules and observed with pollable receipts, not
by exposing an executor.

## World construction

### TECH-017 — Configuration and genesis

Implements: REQ-008, REQ-021, REQ-027, REQ-028, REQ-044

The Bevy entry point is:

```rust
pub struct MoriaPlugin {
    pub config: MoriaConfig,
}

pub struct MoriaClient { /* private shared facade state */ }
pub struct WorldBuilder { /* private unconfigured/unpublished world */ }

pub struct MoriaConfig {
    pub canonical: CanonicalContract,
    pub budgets: ResourceBudgets,
    pub rollback: RollbackConfig,
    pub persistence: PersistenceConfig,
    pub presentation: PresentationConfig,
    pub execution: ExecutionPolicy,
}

pub struct CanonicalContract {
    pub canonical_encoding: ContractDigest,
    pub arithmetic: ContractDigest,
    pub transition: ContractDigest,
    pub collision: ContractDigest,
    pub hash: ContractDigest,
    pub persistence: ContractDigest,
}

pub struct RollbackConfig {
    pub capacity_ticks: u32,
    pub log_ticks: u32,
    pub retain_outcome_bytes: bool,
}

pub struct PersistenceConfig {
    pub default_checkpoint_store: CheckpointStoreId,
    pub replay_sink: ReplaySinkId,
}

pub struct PresentationConfig {
    pub enabled: bool,
    pub show_stale: bool,
    pub chunk_edge_cells: u16, // exactly 32 in v1
    pub maximum_lod: u8,       // 0..=6
}

pub struct WorldGenesisConfig {
    pub placement: PlacementFixedFormat,
}

pub struct ResourceBudgets {
    pub identity: IdentityBudgets,
    pub canonical: CanonicalBudgets,
    pub content: ContentBudgets,
    pub query: QueryBudgets,
    pub observation: ObservationBudgets,
    pub presentation: PresentationBudgets,
    pub checkpoint: CheckpointBudgets,
    pub rollback: RollbackBudgets,
    pub participant: ParticipantBudgets,
    pub runtime: RuntimeBudgets,
}

pub struct IdentityBudgets {
    pub worlds: u32,                         // default 1; max 16
    pub retired_replay_streams_per_client: u32, // default 4; max 64
    pub materials_per_world: u32,            // default 4,096; max 65,535
    pub volumes_per_world: u32,              // default 65,536; max 1,048,576
    pub participants_per_world: u32,         // default 64; max 1,024
    pub input_sources_per_world: u32,        // default 4,096; max 65,535
    pub base_sources_per_world: u32,         // default 256; max 1,024
    pub base_authorities_per_world: u32,     // default 65,536; max 1,048,576
    pub content_blob_stores_per_world: u32,  // default 4; max 16
    pub checkpoint_stores_per_world: u32,    // default 4; max 16
    pub replay_sinks_per_world: u32,         // default 4; max 16
    pub rng_streams_per_participant: u32,    // default 32; max 256
    pub representation_contracts_per_participant: u32, // default 64; max 256
    pub interests_per_world: u32,            // default 4,096; max 16,384
    pub operation_records_per_world: u32,    // default 16,384; max 65,536
    pub terminal_receipts_per_world: u32,    // default 8,192; max 65,536
    pub terminal_receipt_bytes_per_world: u64, // default 64 MiB; max 512 MiB
    pub root_leases_per_world: u32,          // default 4,096; max 16,384
    pub artifact_leases_per_world: u32,      // default 256; max 1,024
}

pub struct CanonicalBudgets {
    pub pending_ticks: u32,                  // exactly 1
    pub inputs_per_tick: u32,                // default/max 4,096
    pub encoded_bytes_per_tick: u64,         // default/max 8 MiB
    pub correlation_bytes_per_tick: u64,     // default/max 320 KiB
    pub bricks_per_command: u32,             // default/max 64
    pub cells_per_command: u32,              // default/max 32,768
    pub changed_bricks_per_tick: u32,        // default 512; max 16,384
    pub scratch_bytes: u64,                  // default 256 MiB; max 1 GiB
}

pub struct ContentBudgets {
    pub base_request_queue: u32,              // default 256; max 1,024
    pub base_requests_in_flight: u32,         // default 32; max 128
    pub base_completion_bytes_in_flight: u64, // default 64 KiB; max 256 KiB
    pub materialization_bricks_per_job: u32,  // default 4,096; max 16,384
    pub resident_dense_bricks: u32,           // default 65,536; max 4,194,304
    pub resident_uniform_bricks: u32,         // default 65,536; max 4,194,304
    pub resident_radix_nodes: u32,            // default 1,048,576; max 16,777,216
    pub resident_directory_buckets: u32,      // default 1,048,576; max 16,777,216
    pub authoritative_gpu_bytes: u64,         // default 2 GiB; max 16 GiB
}

pub struct QueryBudgets {
    pub queued_requests: u32,                 // default 256; max 1,024
    pub in_flight_requests: u32,              // default 3; max 8
    pub bricks_per_request: u32,              // default 4,096; max 16,384
    pub records_per_result: u32,              // default 65,536; max 262,144
    pub bytes_per_result: u64,                // default 4 MiB; max 16 MiB
    pub volume_revisions_per_request: u32,    // default 256; max 1,024
    pub readback_bytes_in_flight: u64,        // default 48 MiB; max 128 MiB
}

pub struct ObservationBudgets {
    pub records_per_world: u32,               // default 8,192; max 65,536
    pub payload_bytes_per_world: u64,         // default 32 MiB; max 256 MiB
    pub bytes_per_record: u32,                // default 4 KiB; max 64 KiB
    pub subscriptions_per_world: u32,         // default 64; max 256
    pub volumes_per_subscription: u32,        // default 1,024; max 16,384
    pub records_per_poll: u32,                // default 256; max 4,096
    pub bytes_per_poll: u64,                  // default 1 MiB; max 16 MiB
    pub resnapshot_volume_summaries: u32,     // default 1,024; max 16,384
    pub resnapshot_region_summaries: u32,     // default 4,096; max 65,536
    pub resnapshot_bytes: u64,                // default 16 MiB; max 256 MiB
}

pub struct PresentationBudgets {
    pub queued_chunks: u32,                   // default 4,096; max 16,384
    pub resident_chunks: u32,                 // default 65,536; max 1,048,576
    pub in_flight_jobs: u32,                  // default 3; max 8
    pub vertices_per_job: u32,                // default 1,048,576; max 4,194,304
    pub indices_per_job: u32,                 // default 6,291,456; max 25,165,824
    pub bytes_per_job: u64,                   // default 64 MiB; max 256 MiB
    pub resident_bytes: u64,                  // default 1 GiB; max 8 GiB
    pub dressing_records_per_chunk: u32,      // default 65,536; max 262,144
}

pub struct CheckpointBudgets {
    pub queued_requests: u32,                 // default 4; max 16
    pub active_requests: u32,                 // exactly 1
    pub staging_slots: u32,                   // default/max 3
    pub mapped_bytes_in_flight: u64,          // default 16 MiB; max 64 MiB
    pub store_bytes_in_flight: u64,           // default 64 MiB; max 256 MiB
    pub bytes_per_blob: u64,                  // default 8 MiB; max 64 MiB
    pub bytes_per_checkpoint: u64,            // default 1 GiB; max 4 GiB
    pub manifest_nodes: u32,                  // default 1,048,576; max 16,777,216
    pub manifest_blobs: u32,                  // default 1,048,576; max 16,777,216
    pub manifest_bytes: u64,                  // default 64 MiB; max 256 MiB
}

pub struct RollbackBudgets {
    pub retained_frontiers: u32,              // default 32; min 20; max 256
    pub retained_bytes: u64,                  // default 2 GiB; max 16 GiB
    pub genesis_persistent_bytes: u64,        // default 256 MiB; max 2 GiB
    pub frontier_metadata_bytes: u64,         // default 2 MiB; max 64 MiB
    pub log_ticks: u32,                       // default 256; max 65,536
    pub log_bytes: u64,                       // default 256 MiB; max 4 GiB
    pub replay_sink_records_in_flight: u32,   // default 64; max 1,024
    pub replay_sink_bytes_in_flight: u64,     // default 64 MiB; max 512 MiB
    pub active_public_replays: u32,           // default 1; max 16
    pub ticks_per_public_replay: u32,         // default 256; max 4,096
    pub bytes_per_public_replay: u64,         // default 1 GiB; max 4 GiB
    pub result_bytes_per_public_replay: u64,  // default 32 MiB; max 512 MiB
    pub divergence_artifact_bytes: u64,       // default 32 MiB; max 256 MiB
    pub active_corrections: u32,              // exactly 1
    pub ticks_per_correction: u32,            // default 256; max 4,096
    pub bytes_per_correction: u64,            // default 1 GiB; max 4 GiB
    pub result_bytes_per_correction: u64,     // default 32 MiB; max 512 MiB
    pub recovery_replay_ticks: u32,           // default 256; max 4,096
}

pub struct ParticipantBudgets {
    pub operations_in_flight: u32,            // default 64; max 1,024
    pub input_bytes_per_tick: u64,             // default/max 8 MiB
    pub effects_per_tick: u32,                 // default/max 4,096
    pub effect_bytes_per_tick: u64,            // default/max 8 MiB
    pub events_per_tick: u32,                  // default 4,096; max 16,384
    pub event_bytes_per_tick: u64,             // default 4 MiB; max 16 MiB
    pub bytes_per_event: u32,                  // default 1 KiB; max 4 KiB
    pub state_and_snapshot_bytes_per_frontier: u64, // default/max 64 MiB
    pub snapshot_bytes_per_checkpoint: u64,    // default/max 64 MiB
    pub artifact_records_per_tick: u32,        // default 1,048,576; max 4,194,304
    pub artifact_bytes_per_tick: u64,          // default 64 MiB; max 256 MiB
}

pub struct RuntimeBudgets {
    pub interest_control_queue: u32,           // default 4,096; max 16,384
    pub callback_completion_slots: u32,        // default 128; max 256
    pub callback_completion_bytes: u64,        // default 128 MiB; max 512 MiB
    pub render_completion_cells: u32,          // exactly 32
}
```

`RollbackConfig` defaults to `{ capacity_ticks: 32, log_ticks: 256,
retain_outcome_bytes: true }` and must equal the corresponding rollback budget
tick fields. Presentation defaults are enabled, stale display allowed, 32-cell
chunks, and maximum LOD 6. The configured replay sink and the per-world
consumer-selected replay stream key passed to `begin_world` are mandatory, and
the frozen registered sink descriptor must fit the replay-sink count/byte
limits. Moria never derives, randomizes, or replaces the stream key.
`WorldGenesisConfig.placement` is also mandatory and frozen by `begin_world`;
TECH-007 defines its validation and fingerprint participation. Different
worlds hosted by one client may select different valid formats, but no
canonical value or artifact may cross between them without an explicit
consumer conversion outside Moria.

`MoriaPlugin` installs one `MoriaClient` resource and feature plugins. A
consumer constructs exactly one world through `WorldBuilder`; multiple worlds
within one Bevy app are isolated by `WorldId`, queues, roots, and budgets.

```rust
impl WorldBuilder {
    pub fn register_material(&mut self, def: MaterialDefinition)
        -> Result<(), ConfigError>;
    pub fn register_input_source(&mut self, def: InputSourceRegistration)
        -> Result<(), ConfigError>;
    pub fn register_base_source(&mut self, source: Arc<dyn BaseContentSource>)
        -> Result<(), ConfigError>;
    pub fn register_base_authority(&mut self, def: BaseAuthorityRegistration)
        -> Result<(), ConfigError>;
    pub fn register_content_blob_store(&mut self, store: Arc<dyn ContentBlobStore>)
        -> Result<(), ConfigError>;
    pub fn register_checkpoint_store(&mut self, store: Arc<dyn CheckpointStore>)
        -> Result<(), ConfigError>;
    pub fn register_replay_sink(&mut self, sink: Arc<dyn ReplaySink>)
        -> Result<(), ConfigError>;
    pub fn register_volume(&mut self, def: GenesisVolume)
        -> Result<(), ConfigError>;
    pub fn register_participant(&mut self, adapter: ParticipantRegistration)
        -> Result<(), ConfigError>;
}

pub struct InputSourceRegistration {
    pub id: InputSourceId,
    pub schema: SchemaDigest,
    pub max_inputs_per_tick: u32,
    pub max_encoded_bytes_per_tick: u32,
}

pub struct BaseAuthorityRegistration {
    pub id: BaseAuthorityId,
    pub authority: BaseAuthority,
}

pub enum ParticipantRegistration {
    Cpu(Arc<dyn CanonicalParticipant>),
    Gpu(Arc<dyn BevyGpuParticipant>),
}
```

Builder calls only construct private configuration. `publish_genesis` freezes
registries, checks all IDs/domains/limits/content proofs/participant strategies,
validates the runtime device capabilities and canonical-math implementation,
materializes the configured
genesis-resident set, calculates canonical genesis bytes and root, durably
exports the genesis replay header, and then publishes the explicit pre-tick
`FrontierPosition::Genesis`. The resulting world is ready **for** tick zero;
genesis does not confirm or consume a numbered tick. Any error leaves no usable
world or partial registry.
There is no default content, material, participant strategy, RNG seed,
placement format, or empty-world substitution.

Every registered descriptor supplies its own stable ID; Moria never assigns a
provider ID. Duplicate IDs within one registry return
`ConfigErrorCode::DuplicateId` without replacing the original entry. The
namespaces are distinct, but every reference is type checked: each genesis
volume's `BaseAuthorityId` must resolve; every reconstructable authority's
source and every bundled authority's content store must resolve; every
canonical `InputHeader.source` must name an `InputSourceRegistration`; the
configured default checkpoint store and replay sink must resolve; and
every participant descriptor must match its registration kind. Freeze also
checks provider descriptor IDs, contracts, per-provider bounds, and all
aggregate budget sums. Missing, extra-kind, or contract-mismatched references
fail before a callback or device allocation. After freeze no registry can be
modified or replaced.
The pair `(PersistenceConfig.replay_sink, WorldBuilder.replay_stream)` is
frozen and reserved when a builder operation—genesis, durable restore, or
public replay—is accepted, and must be unique among all constructing,
published, or retired stream reservations owned by the same `MoriaClient`.
When multiple private builders select the same pair, the first accepted
operation proceeds and every later call returns its operation-specific
rejection: genesis uses `ConfigErrorCode::DuplicateId`, while restore/replay
use `AdmissionCode::DuplicateReplayStream`; no sink is called and the
unchanged builder/request is returned. Moria cannot detect reuse by another process or client, so
cross-process uniqueness remains the consumer's responsibility and a sink may
reject such reuse as `StoreErrorCode::Corrupt`.
Before accepting any of those three publication operations, Moria also
reserves one `identity.retired_replay_streams_per_client` permit for the
stream's eventual tombstone. Exhaustion returns
`ConfigErrorCode::RetiredReplayStreamCapacity` for genesis or
`AdmissionCode::RetiredReplayStreamCapacity` for restore/replay, with the
identity budget field supplied by the config diagnostic/admission context,
and the unchanged builder/request; identity field codes follow
`IdentityBudgets` declaration order starting at one. No sink call occurs.
Failure before the route's sequence-zero sink call releases both
reservations. Once that call is invoked, the tombstone
permit is committed for the `MoriaClient` lifetime: the reservation is
`Active` while a world exists and becomes `Retired` on normal shutdown or
construction failure, so that pair can never restart at sequence zero.
Active/retired stream reservations do not count against `identity.worlds`.
An accepted genesis/restore/replay construction failure releases its world
permit before the receipt becomes terminal, so at the default `worlds == 1`
a new builder using a different stream remains admissible while a pair whose
sink was invoked remains rejected.
The separate default of four stream tombstones permits that retry and bounds
sequential recreation; a fifth sequence-zero attempt in one client fails with
the named capacity error before consumer code.
`PersistenceConfig.default_checkpoint_store` is the ID used by the public
request-construction helpers and must resolve, but every admitted checkpoint,
restore, shutdown checkpoint, and recovery request carries and binds its exact
store ID; failure never falls through to the default or another store.

Pre-admission construction rejection returns the unchanged private builder
and, for restore/replay, the unchanged request in `GenesisRejected`,
`RestoreRejected`, or `ReplayRejected`. After acceptance, the corresponding
receipt remains terminal once ready/failed/cancelled; TECH-021 and TECH-047
define the required replay-stream bootstrap that gates each ready
publication.

The budget comments above are normative defaults and portable compiled
maxima. `authoritative_gpu_bytes`, `presentation.resident_bytes`, and
`rollback.retained_bytes` are additionally capped by checked page arithmetic
and the selected adapter's granted allocation limits. TECH-036 defines all
cross-field validation; exceeding a compiled maximum or failing a cross-limit
check rejects configuration before any consumer callback or GPU allocation.
`ResourceBudgets::default()` is required to produce exactly these
self-consistent values and passes TECH-036's 20-frontier inequality.
`MoriaConfig` deliberately has no blanket `Default`: provider IDs, canonical
contracts, execution policy, and each world's placement format require
explicit consumer authority.

### TECH-018 — Materials and volumes

Implements: REQ-003, REQ-008, REQ-013, REQ-019, REQ-020

```rust
pub struct MaterialDefinition {
    pub id: MaterialId,
    pub occupancy: OccupancyClass,
    pub presentation: MaterialPresentation,
}

pub struct AssetHandleId(u64);

pub struct MaterialPresentation {
    pub surface: SurfaceStyle,
    pub material_asset: AssetHandleId,
    pub dressing_asset: Option<AssetHandleId>,
}

pub enum SurfaceStyle {
    SmoothDensity,
    CrispCell,
}

pub enum OccupancyClass {
    SolidAbove { density_q8_8: i16 },
    Never,
}

pub enum VolumeKind {
    Static,
    Dynamic,
}

pub struct GenesisVolume {
    pub requested_id: Option<VolumeId>,
    pub kind: VolumeKind,
    pub domain: LocalCellAabb,
    pub pivot: LocalCellPoint,
    pub placement: PlacementQ,
    pub base: BaseAuthorityId,
}
```

`AssetHandleId` is an opaque consumer asset-registry key with infallible
`from_raw(u64)` and `get(self) -> u64`; Moria assigns no meaning to zero.

Material presentation contains only surface style and asset handles required
by Moria's rendering; arbitrary consumer metadata stays outside Moria.
`SolidAbove` requires a threshold from 1 through `i16::MAX`; `Never` is
inspectable material that does not contribute occupancy. Occupancy class is
canonical material metadata, while asset handles and surface style are
derived-presentation configuration. Neither is physics or damage data.

Static and dynamic volumes use identical query, mutation, checkpoint, and
presentation methods. Only dynamic volumes accept `SetPlacement`. World-space
overlap returns all volume encounters in stable `VolumeId` order. Moria never
merges volumes or selects a response.

### TECH-070 — Complete consumer facade

Implements: REQ-002, REQ-004, REQ-005, REQ-009, REQ-010, REQ-012, REQ-014, REQ-015, REQ-021, REQ-022, REQ-035, REQ-044

The following is the complete v1 callable surface. Methods not present here or
on the registered content/store/participant traits are not consumer
capabilities. In particular, there is no direct storage, renderer-buffer,
unticked mutation, participant scheduler, or privileged determinism facade.

```rust
pub struct Rejected<T> {
    pub request: T,
    pub reason: AdmissionError,
}

pub struct RestoreRejected {
    pub builder: WorldBuilder,
    pub request: RestoreRequest,
    pub reason: AdmissionError,
}

pub struct ReplayRejected {
    pub builder: WorldBuilder,
    pub request: ReplayRequest,
    pub reason: AdmissionError,
}

pub struct GenesisRejected {
    pub builder: WorldBuilder,
    pub reason: ConfigError,
}

impl MoriaClient {
    pub fn begin_world(
        &self,
        id: WorldId,
        replay_stream: ReplayStreamKey,
        genesis: WorldGenesisConfig,
    )
        -> Result<WorldBuilder, ConfigError>;
    pub fn reserve_tick(
        &self,
        world: WorldId,
        tick: Tick,
        limits: TickReservation,
    ) -> Result<TickPermit, ReserveError>;
    pub fn submit_tick(
        &self,
        batch: SealedTickBatch,
    ) -> Result<TickReceipt, Rejected<SealedTickBatch>>;

    pub fn upsert_interest(
        &self,
        request: InterestRequest,
    ) -> Result<InterestReceipt, Rejected<InterestRequest>>;
    pub fn withdraw_interest(
        &self,
        request: InterestWithdrawal,
    ) -> Result<InterestReceipt, Rejected<InterestWithdrawal>>;

    pub fn submit_query(
        &self,
        request: QueryRequest,
    ) -> Result<QueryReceipt, Rejected<QueryRequest>>;

    pub fn subscribe_observations(
        &self,
        request: ObservationSubscriptionRequest,
    ) -> Result<ObservationSubscription, Rejected<ObservationSubscriptionRequest>>;
    pub fn request_observation_resnapshot(
        &self,
        subscription: &ObservationSubscription,
        request: ObservationResnapshotRequest,
    ) -> Result<ObservationResnapshotReceipt, Rejected<ObservationResnapshotRequest>>;

    pub fn request_checkpoint(
        &self,
        request: CheckpointRequest,
    ) -> Result<CheckpointReceipt, Rejected<CheckpointRequest>>;
    pub fn request_correction(
        &self,
        request: CorrectionRequest,
    ) -> Result<CorrectionReceipt, Rejected<CorrectionRequest>>;
    pub fn request_recovery(
        &self,
        request: RecoveryRequest,
    ) -> Result<RecoveryReceipt, Rejected<RecoveryRequest>>;

    pub fn telemetry(
        &self,
        world: WorldId,
    ) -> Result<TelemetrySnapshot, TelemetryError>;
    pub fn shutdown(
        &self,
        request: ShutdownRequest,
    ) -> Result<ShutdownReceipt, Rejected<ShutdownRequest>>;
}

impl WorldBuilder {
    pub fn publish_genesis(self) -> Result<GenesisReceipt, GenesisRejected>;
    pub fn restore_checkpoint(
        self,
        request: RestoreRequest,
    ) -> Result<RestoreReceipt, RestoreRejected>;
    pub fn replay_records(
        self,
        request: ReplayRequest,
    ) -> Result<ReplayReceipt, ReplayRejected>;
}

impl ObservationSubscription {
    pub fn poll(&mut self, limits: ObservationPollLimits) -> ObservationPoll;
    pub fn resume(&mut self, cursor: ObservationCursor)
        -> Result<(), ObservationCursorError>;
    pub fn close(&mut self) -> ObservationCloseResult;
}
```

The facade's common ready/result records are:

```rust
pub struct FrontierSummary {
    pub world: WorldId,
    pub position: FrontierPosition,
    pub root_hash: CanonicalHash,
    pub status: AuthorityStatus,
}

pub enum FrontierPosition {
    Genesis,
    Confirmed(Tick),
}

impl FrontierPosition {
    pub fn next_tick(self) -> Option<Tick>;
}

impl FrontierSummary {
    pub fn next_tick(&self) -> Option<Tick>;
}

pub enum AuthorityStatus {
    ReplayGrade,
    DiagnosticCandidate,
}

pub struct GenesisReady {
    pub frontier: FrontierSummary, // position is exactly Genesis
    pub next_tick: Tick,           // exactly Tick::from_raw(0)
    pub replay: ReplayStreamPosition,
}

pub struct TickReservation {
    pub max_inputs: u32,
    pub max_encoded_bytes: u64,
    pub max_correlation_bytes: u64,
}

pub struct VolumeAffectedBounds {
    pub volume: VolumeId,
    pub local: LocalCellAabb,
    pub world: WorldAabbQ,
}

pub struct AffectedBounds {
    pub volumes: BoundedVec<VolumeAffectedBounds>,
}

pub enum InterestChange {
    Upserted,
    Withdrawn,
}

pub struct InterestCoverage {
    pub volume: VolumeId,
    pub requested: WorldAabbQ,
    pub covered: WorldAabbQ,
    pub complete: bool,
}

pub struct InterestApplied {
    pub world: WorldId,
    pub interest: InterestId,
    pub change: InterestChange,
    pub coverage: BoundedVec<InterestCoverage>,
}

pub struct QueryResult {
    pub frontier: FrontierSummary,
    pub revisions: BoundedVec<VolumeRevisionFact>,
    pub inspected: BoundedVec<InspectedRange>,
    pub missing: BoundedVec<MissingRange>,
    pub freshness: QueryFreshnessResult,
    pub data: QueryData,
}

pub struct VolumeRevisionFact {
    pub volume: VolumeId,
    pub revision: VolumeRevision,
}

pub struct InspectedRange {
    pub volume: VolumeId,
    pub local: LocalCellAabb,
    pub world: WorldAabbQ,
}

pub struct MissingRange {
    pub volume: VolumeId,
    pub local: LocalCellAabb,
    pub reason: AvailabilityCode,
}

pub enum QueryFreshnessResult {
    Met,
    Stale { unmet: BoundedVec<MinimumVolumeRevision> },
}

pub enum QueryData {
    Samples(BoundedVec<MaterialFact>),
    Region(BoundedVec<MaterialFact>),
    Occupancy(BoundedVec<CollisionFact>),
    Trace(BoundedVec<CollisionFact>),
    Overlap(BoundedVec<CollisionFact>),
    Sweep(BoundedVec<CollisionFact>),
}

pub struct MaterialFact {
    pub volume: VolumeId,
    pub local: LocalCellPoint,
    pub cell: CellWire,
}

pub struct Recovered {
    pub frontier: FrontierSummary,
    pub old_generation: DeviceGeneration,
    pub new_generation: DeviceGeneration,
    pub replayed_ticks: u32,
}

pub struct DirtyRootSummary {
    pub frontier: FrontierSummary,
    pub dirty_scar_keys: u32,
    pub dirty_metadata_keys: u32,
}

pub struct ShutdownReport {
    pub last_frontier: FrontierSummary,
    pub last_durable: Option<DurableFrontier>,
    pub abandoned_receipts: BoundedVec<ReceiptId>,
    pub dirty_roots: BoundedVec<DirtyRootSummary>,
    pub checkpoint: Option<CheckpointCommitted>,
    pub replay_export_failure: Option<ReplayExportFailure>,
}

pub struct TelemetryCounter {
    pub key: TelemetryKey,
    pub current: u64,
    pub capacity: u64,
    pub high_water: u64,
}

pub enum TelemetryKey {
    Identity(BudgetMetric),
    Canonical(BudgetMetric),
    Content(BudgetMetric),
    Query(BudgetMetric),
    Observation(BudgetMetric),
    Presentation(BudgetMetric),
    Checkpoint(BudgetMetric),
    Rollback(BudgetMetric),
    Participant(BudgetMetric),
    Runtime(BudgetMetric),
}

pub enum BudgetMetric {
    Records,
    Bytes,
    InFlight,
    Queued,
    OldestAgeMicros,
    CompletionMicros,
}

pub struct FailureCounter {
    pub code: ErrorCode,
    pub count: u64,
}

pub struct TelemetrySnapshot {
    pub frontier: FrontierSummary,
    pub durable: Option<DurableFrontier>,
    pub device_generation: DeviceGeneration,
    pub execution: ExecutionSummary,
    pub counters: BoundedVec<TelemetryCounter>,
    pub failures: BoundedVec<FailureCounter>,
}
```

`FrontierPosition` is canonically encoded with tag `0 = Genesis` and
`1 = Confirmed` followed by the little-endian tick. No sentinel `Tick` value
represents genesis. `FrontierSummary::next_tick()` is the checked rule
`Genesis -> Some(Tick::from_raw(0))`,
`Confirmed(t) -> t.get().checked_add(1).map(Tick::from_raw)`, and is the only
eligibility calculation used by reservation, submission, restore, replay,
correction, persistence, and observation code. A `TickConfirmed.frontier`
always has `Confirmed(the submitted tick)`. A `GenesisReady.frontier` always
has `Genesis`; it may be queried and hashed but is not a confirmed-tick
rollback entry. `ShutdownReport.last_frontier` may therefore be `Genesis` when
the world shuts down before confirming tick zero.

`ExecutionSummary` is the closed record in TECH-040; telemetry contains no
string or dynamic extension map. `CollisionFact` is the closed canonical wire record in
TECH-052. All vectors above reserve their request/configuration maximum before
admission; `QueryData` must match the admitted `QueryKind`, and every
`InspectedRange`/`MissingRange` partitions only the requested scope. There is
no generic material-result payload or backend-owned collection.

Every accepted asynchronous method creates exactly one operation record and
receipt ID. Every `Rejected<T>` returns the exact owned request unchanged;
`GenesisRejected`/`RestoreRejected` return the still-private builder and the
unchanged restore request where applicable. Copying an ID in a request does
not transfer ownership of the referenced world or interest. Accepted requests
are copied into Moria-owned bounded storage, so the caller may drop its receipt
without losing the operation. No facade method blocks on GPU, I/O,
participant, or callback completion.

`TelemetrySnapshot` is the only synchronous read above. It copies the latest
already-collected bounded counters from the main world and returns the closed
`TelemetryError::WorldUnknown`, `TelemetryError::WorldClosed`, or
`TelemetryError::TelemetryBusy` variants from TECH-027; it never drives
progress, maps a buffer, waits, or pins a root. Its encoded size is bounded by
the identity and operation-record budgets. All other result-producing calls
use the receipts below.

The public newtypes and bounded owners used by normative signatures are
closed, non-placeholder types: IDs, ticks, revisions, hashes, digests,
fixed-point points/vectors/AABBs, and limits are fixed-width value types;
`OwnedBytes`, `BoundedBytes64`, `BoundedUtf8<N>`, and `BoundedVec<T>` own
validated finite allocations whose capacity is fixed at construction and
counted against the accepting operation; request/result/error types are the
closed records or enums defined in their owning `TECH` contract. `Arc`,
`Vec`, `Result`, `Option`, and `Box` have their standard Rust meanings. A
public Rust signature may not introduce another named type without defining
its ownership, bound, and owner contract in this document.

```rust
pub struct BoundedVec<T> { /* private owned allocation and fixed capacity */ }
pub struct BoundedBytes { /* private owned allocation, admitted u32 capacity */ }
pub struct BoundedBytes64([u8; 64], u8);
pub struct BoundedUtf8<const N: usize> { /* private validated bytes, length <= N */ }
pub struct OwnedBytes { /* private immutable fallibly allocated bytes plus admitted u64 length */ }

pub enum BoundedOwnerError {
    CapacityTooLarge,
    LengthExceedsCapacity,
    InvalidUtf8,
    AllocationFailed,
}

pub struct VecConstructionRejected<T> {
    pub values: Vec<T>,
    pub reason: BoundedOwnerError,
}
pub struct BytesConstructionRejected {
    pub bytes: Vec<u8>,
    pub reason: BoundedOwnerError,
}
pub struct BoundedPushRejected<T> {
    pub value: T,
    pub reason: BoundedOwnerError,
}

impl<T> BoundedVec<T> {
    pub fn try_with_capacity(capacity: u32) -> Result<Self, BoundedOwnerError>;
    pub fn try_from_vec(
        values: Vec<T>,
        capacity: u32,
    ) -> Result<Self, VecConstructionRejected<T>>;
    pub fn try_push(&mut self, value: T) -> Result<(), BoundedPushRejected<T>>;
    pub fn as_slice(&self) -> &[T];
    pub fn iter(&self) -> std::slice::Iter<'_, T>;
    pub fn len(&self) -> u32;
    pub fn capacity(&self) -> u32;
    pub fn is_empty(&self) -> bool;
    pub fn into_vec(self) -> Vec<T>;
}

impl BoundedBytes {
    pub fn try_with_capacity(capacity: u32) -> Result<Self, BoundedOwnerError>;
    pub fn try_from_vec(
        bytes: Vec<u8>,
        capacity: u32,
    ) -> Result<Self, BytesConstructionRejected>;
    pub fn try_extend_from_slice(&mut self, bytes: &[u8])
        -> Result<(), BoundedOwnerError>;
    pub fn as_slice(&self) -> &[u8];
    pub fn len(&self) -> u32;
    pub fn capacity(&self) -> u32;
    pub fn is_empty(&self) -> bool;
    pub fn into_vec(self) -> Vec<u8>;
}

impl BoundedBytes64 {
    pub fn try_from_slice(bytes: &[u8]) -> Result<Self, BoundedOwnerError>;
    pub fn as_slice(&self) -> &[u8];
    pub fn len(&self) -> u8;
    pub fn is_empty(&self) -> bool;
}

impl<const N: usize> BoundedUtf8<N> {
    pub fn try_from_bytes(bytes: Vec<u8>)
        -> Result<Self, BytesConstructionRejected>;
    pub fn as_str(&self) -> &str;
    pub fn as_bytes(&self) -> &[u8];
    pub fn len(&self) -> usize;
    pub fn is_empty(&self) -> bool;
    pub fn into_bytes(self) -> Vec<u8>;
}

impl OwnedBytes {
    pub fn try_from_vec(
        bytes: Vec<u8>,
        max_bytes: u64,
    ) -> Result<Self, BytesConstructionRejected>;
    pub fn as_slice(&self) -> &[u8];
    pub fn len(&self) -> u64;
    pub fn is_empty(&self) -> bool;
}
```

All lengths and capacities are exact and never saturate. `try_from_vec`
returns the original allocation on failure, `try_push` returns the rejected
element, and `try_extend_from_slice` performs no partial append.
`BoundedBytes64` rejects a stored length above 64; `BoundedUtf8` rejects
invalid UTF-8 and a length above `N`. `OwnedBytes` has no spare growable
capacity in the public contract: `max_bytes` is checked at construction,
`len()` is the exact immutable length, and successful construction stores the
bytes in an immutable, shareable allocation. Its concrete backing owner remains
private so the implementation can use a stable-Rust fallible allocation path;
the facade never promises conversion to a standard owner whose construction
would require a second unchecked allocation. Failed construction returns the
caller's original `Vec<u8>` unchanged.

Caller-side construction performs only checked finite allocation. When one of
these owners enters a facade request, provider callback, or completion sink,
that accepting contract separately validates its exact length and declared
capacity and reserves the corresponding Moria budget before ownership
transfers. No public method grows a bounded owner past its immutable capacity;
decoding validates count, byte length, and trailing data before allocation.

The mechanical public-type index is:

| Names | Resolution and owner |
| --- | --- |
| `WorldId`, `MaterialId`, `VolumeId`, `ParticipantId`, `InputSourceId`, `RngStreamId`, `BaseContentSourceId`, `BaseAuthorityId`, `ContentBlobStoreId`, `CheckpointStoreId`, `ReplaySinkId`, `BaseRequestId`, `InterestId`, `CorrelationId`, `ObservationStreamId`, `ReceiptId`, `DeviceGeneration`, `Tick`, `VolumeRevision`, `CanonicalOrder`, `AssetHandleId`, `CheckpointKey`, `ReplayStreamKey`, `ContentLineage` | private-field fixed-width copy newtypes with the exact validating/infallible constructor and accessor families in TECH-005/017/018/021/022/025/037/041/043/047; no tuple construction |
| `CanonicalHash`, `ContentDigest`, `ContractDigest`, `SchemaDigest`, `BlobDigest` | distinct private-field 32-byte digest newtypes with lossless byte constructors/accessors; TECH-005/008/009/041/043 |
| `SimulationUnitId`, `PlacementFixedFormat`, `PlacementScalar`, `TurnQ32`, `WorldPointQ`, `WorldVectorQ`, `WorldAabbQ`, `LocalCellPoint`, `LocalCellAabb`, `BrickCoord`, `SegmentQ`, `PlacementQ`, `QuatQ14`, `CellWire` | distinct fixed-width canonical values and per-world placement format; TECH-006/007/018/051/071 |
| `MoriaClient`, `WorldBuilder`, every `*Permit`, `*Receipt`, `ObservationSubscription`, and every participant/root/artifact lease or state token | opaque generational handles; their owning operation contract defines clone/drop/pin behavior, and no handle exposes storage beyond the explicit participant-only views in TECH-029/054 |
| `BoundedVec<T>`, `BoundedBytes`, `BoundedBytes64`, `BoundedUtf8<N>`, `OwnedBytes`, `BoundedOwnerError`, `VecConstructionRejected<T>`, `BytesConstructionRejected`, `BoundedPushRejected<T>` | callable finite owners and lossless construction failures; TECH-070 and the accepting resource contract |
| `ParticipantTokenMetadata`, `ParticipantReplayRecordView`, `ColliderArtifactView`, `SnapshotMetadata`, `PreparedParticipantState` | immutable bounded participant-owned state/replay/collider views and staged state record; TECH-016/029/053 |
| `CanonicalContract`, `RollbackConfig`, `PersistenceConfig`, `PresentationConfig`, `WorldGenesisConfig`, `ExecutionPolicy`, `CandidateDiagnostics`, `ParticipantRepresentationContract`, `TickReservation`, `InterestCapacity`, `QueryCapacity`, `MinimumRevisionGap`, `BlobLimits`, `RestoreLimits`, every `*Descriptor`, and every `*Limits` | closed configuration/request and required/supported capacity/freshness records; TECH-016/017/019/022/023/029/036/040/041/043/046/054 |
| `CanonicalInput` and its variant payloads, `QueryKind`, `QueryScope`, `CollisionShapeQ`, `VolumeSelector`, `VolumeKind`, `ReplayAppendRange`, participant strategy/failure policy, and all lifecycle/poll/start/persistence policy enums | closed tagged enums in TECH-018/020/022/023/025/028/029/030/041/047/051 |
| `QueryResult`, `Observation`, `ObservedVolumeSummary`, `ObservedRegionSummary`, `TelemetrySnapshot`, `ExecutionSummary`, `FrontierSummary`, `FrontierPosition`, `ReplayStreamPosition`, `ReplayExportFailure`, and every receipt `Ready` payload | the concrete bounded records in TECH-020/022/025/026/038/040/045-048/070; their worst-case bytes are reserved before admission |
| `OperationProgress`, `ProgressBlocker`, `QueryReadinessReason`, `NewtypeValueError`, `ConfigError`, `AdmissionError`, `AdmissionContext`, `ReserveError`, `PushError`, `BatchError`, `CanonicalFailure`, `FailedNoAdvance`, `TickNoAdvanceCause`, `TelemetryError`, and every operation-specific `*Error`/`*Unavailable` | closed typed progress/errors under TECH-005/021/023/027; query progress and unavailability retain exact bounded blockers, tick-global failure retains attempted tick/source/cause, telemetry has exact synchronous variants, and ordinary operation errors retain scope, retryability, and committed-effect fields |
| `BaseBrickCompletion`, participant completion/effect/event/state/snapshot sinks, checkpoint/content-store sinks, and `ReplayAppendSink` | non-clone Moria-owned bounded completion tokens; TECH-016/029/036/041/043/045/047/054 |
| Bevy/wgpu names in `moria::bevy::gpu_participant` | deliberately coupled borrowed adapter types, generation-scoped and never general-facade or durable types; TECH-003/031/054 |

Capitalized words that are enum variants, generic parameters, standard-library
types, or units (`KiB`, `MiB`, `GiB`) are not public named types. This index and
the snippets are checked together; duplicate struct fields, an indexed name
with no owner row, or a facade request/receipt not reachable through TECH-070
is a validation failure.

## Tick admission and completion

### TECH-019 — Sealed tick batch and permit

Implements: REQ-005, REQ-011, REQ-017, REQ-027, REQ-033

```rust
pub struct TickPermit { /* private unique reservation and owned inputs */ }
pub struct SealedTickBatch { /* private immutable canonical bytes/sidecar */ }

pub struct PushRejected {
    pub input: CanonicalInput,
    pub correlation: Option<CorrelationMetadata>,
    pub reason: PushError,
}

pub struct SealRejected {
    pub permit: TickPermit,
    pub reason: BatchError,
}

impl TickPermit {
    pub fn push(
        &mut self,
        input: CanonicalInput,
        correlation: Option<CorrelationMetadata>,
    ) -> Result<(), PushRejected>;
    pub fn seal(self) -> Result<SealedTickBatch, SealRejected>;
}

```

Reservation atomically claims bounded queue bytes and one pending-tick slot;
dropping an unused permit releases them without input loss. `seal` canonical-
encodes, sorts, detects duplicate keys, verifies declared counts, and consumes
the builder on success. `push` rejection returns the input and correlation;
`seal` rejection returns the permit with all inputs still owned so the caller
may correct or drop it. A sealed batch owns immutable canonical bytes, its BLAKE3 digest,
the unforgeable reservation token, and a separately bounded noncanonical
correlation sidecar keyed by the resulting `CanonicalOrder`.

Only `frontier.next_tick()` is accepted: a genesis frontier expects tick zero,
and a confirmed tick `t` expects `t + 1`; `Rejected.reason.code`
classifications are
`WrongWorld`, `BeforeNextTick`, `AfterNextTick`, `AlreadyPending`,
`WorldNotReady`, `DependencyNotReady`, `Full`, `Closed`, and `InvalidBatch`.
`BeforeNextTick`/`AfterNextTick` carry the supplied and expected-next ticks in
`AdmissionContext::TickEligibility`; `InvalidBatch` carries
`AdmissionContext::InvalidBatch`. Rejection returns the owned batch.
Accepted work gets one monotonically increasing noncanonical `ReceiptId`.
Dropping a receipt never cancels admitted work.

### TECH-020 — Canonical inputs and command outcomes

Implements: REQ-011, REQ-017, REQ-028, REQ-031, REQ-033

```rust
pub enum CanonicalInput {
    CreateVolume(CreateVolume),
    RetireVolume(RetireVolume),
    ActivateRegion(ActivateRegion),
    DeactivateRegion(DeactivateRegion),
    SetPlacement(SetPlacement),
    Erase(Erase),
    Place(Place),
    Patch(Patch),
    ParticipantInput(ParticipantInput),
}

pub struct InputHeader {
    pub source: InputSourceId,
    pub sequence: u32,
    pub expected_volume_revision: Option<VolumeRevision>,
    pub expected_source_hash: Option<CanonicalHash>,
}

pub struct CreateVolume {
    pub header: InputHeader,
    pub requested_id: Option<VolumeId>,
    pub kind: VolumeKind,
    pub domain: LocalCellAabb,
    pub pivot: LocalCellPoint,
    pub placement: PlacementQ,
    pub base: BaseAuthorityId,
}

pub struct RetireVolume {
    pub header: InputHeader,
    pub volume: VolumeId,
}

pub struct ActivateRegion {
    pub header: InputHeader,
    pub volume: VolumeId,
    pub local_bricks: BrickAabb,
    pub base_root: ContentDigest,
    pub manifest_subtree: Option<ContentDigest>,
    pub activity_class: u32,
}

pub struct DeactivateRegion {
    pub header: InputHeader,
    pub volume: VolumeId,
    pub local_bricks: BrickAabb,
    pub activity_class: u32,
}

pub struct SetPlacement {
    pub header: InputHeader,
    pub volume: VolumeId,
    pub placement: PlacementQ,
}

pub struct Erase {
    pub header: InputHeader,
    pub volume: VolumeId,
    pub target: MutationShapeQ,
    pub mode: EraseMode,
}

pub struct Place {
    pub header: InputHeader,
    pub volume: VolumeId,
    pub target: MutationShapeQ,
    pub cell: CellWire,
}

pub struct LocalPointQ(pub [PlacementScalar; 3]);

pub enum MutationShapeQ {
    CellAabb(LocalCellAabb),
    Sphere { center: LocalPointQ, radius: PlacementScalar },
    Stamp {
        bounds: LocalCellAabb,
        mask_bits: BoundedBytes,
    },
}

pub enum EraseMode {
    SetEmpty,
    SubtractDensity { amount_q8_8: i16 },
}

pub struct PatchCell {
    pub local: LocalCellPoint,
    pub cell: CellWire,
}

pub struct Patch {
    pub header: InputHeader,
    pub volume: VolumeId,
    pub cells: BoundedVec<PatchCell>,
}

pub struct ParticipantInput {
    pub header: InputHeader,
    pub participant: ParticipantId,
    pub schema: SchemaDigest,
    pub payload: BoundedBytes,
}

pub struct CorrelationMetadata {
    pub id: CorrelationId,       // consumer-selected 128-bit value
    pub payload: BoundedBytes64, // 0..=64 uninterpreted bytes
}
```

`BrickAabb` is a fixed six-`i32` half-open brick-coordinate record. Every
variant has a closed `u8` wire tag and explicit maximum encoded size.
`Stamp.mask_bits` has exactly `ceil(bounds.cell_count / 8)` bytes and uses the
same x-major/y/z bit order as brick cells; unused high bits are zero.
`SubtractDensity.amount_q8_8` must be positive and follows TECH-012's explicit
empty saturation.
Convenience helpers only construct these values; they cannot submit or mutate.
No `Restore`, arbitrary callback, raw shader, raw buffer, or test mutation is a
canonical input. Correlation is explicitly not part of `InputHeader` or
canonical input bytes. At `seal`, it is associated with the input's unique
canonical key and then its sorted `CanonicalOrder`; it cannot affect sorting,
validation, outcomes, hashes, or participant input. Its bytes count against
the tick reservation and observation byte budgets.

```rust
pub enum CommandOutcome {
    Applied {
        tick: Tick,
        order: CanonicalOrder,
        affected: AffectedBounds,
        revision: Option<VolumeRevision>,
        root_hash: CanonicalHash,
    },
    Failed {
        tick: Tick,
        order: CanonicalOrder,
        reason: CanonicalFailure,
        state_unchanged: bool,
    },
    NoOp { tick: Tick, order: CanonicalOrder },
}

pub struct CommandOutcomeView {
    pub canonical: CommandOutcome,
    pub correlation: Option<CorrelationMetadata>,
}

pub struct TickConfirmed {
    pub frontier: FrontierSummary,
    pub outcomes: BoundedVec<CommandOutcomeView>,
    pub participant_events: BoundedVec<ParticipantEvent>,
    pub participant_event_digest: CanonicalHash,
}
```

Failures such as absent IDs, wrong kind, stale revision/hash, invalid cells,
out-of-domain shape, arithmetic overflow, logical capacity, and participant
effect validation have stable wire tags. A command failure may coexist with
other command outcomes in a confirmed tick because it is itself deterministic;
a tick-global fault produces no confirmed outcome list.

Receipt and observation APIs return
`CommandOutcomeView { canonical: CommandOutcome, correlation:
Option<CorrelationMetadata> }`. The canonical outcome encoding contains no
correlation bytes. The sidecar lives until both the terminal receipt cache and
the corresponding observation-ring record expire. An observation gap may
therefore lose correlation and reports that fact; bounded resnapshot does not
reconstruct it. `moria-replay-v1` intentionally omits correlation, so replayed
outcomes carry `None`. Consumers that need correlation after replay retain
their own mapping keyed by `(tick, CanonicalOrder)`.

`TickConfirmed.participant_events` is bounded by the participant aggregate
count/byte budgets and sorted by `(ParticipantId, local_sequence)`. It is the
only live facade delivery of participant-owned event payloads. Dropping the
receipt releases that bounded result permit; TECH-021 backpressures later
admission if consumers retain all terminal result capacity.

### TECH-021 — Receipt lifecycle and cancellation

Implements: REQ-005, REQ-011, REQ-015, REQ-017, REQ-021

All receipt types support nonblocking `poll(&self) -> ReceiptState<T, E>` and a
Bevy `MessageReader` notification. They are `Clone + Send + Sync`; polling is
idempotent. Terminal results remain available while any receipt handle exists;
after the last handle drops, the bounded terminal cache may retain the record
until its count/byte eviction policy expires it.

```rust
pub enum ReceiptState<T, E> {
    Pending(OperationProgress),
    Ready(Arc<T>),
    Failed(Arc<E>),
    Cancelled(CancelledOperation),
}

pub struct OperationProgress {
    pub phase: OperationPhase,
    pub blocker: Option<ProgressBlocker>,
}

pub enum ProgressBlocker {
    Query(QueryReadinessReason),
}

pub enum OperationPhase {
    Verifying,
    Queued,
    Applying,
    WaitingForReadiness,
    Pinning,
    Loading,
    LoadingOwnedRecords,
    VerifyingHeader,
    ExportingReplayHeader,
    ExportingReplayPrefix,
    ExportingCorrectionBranch,
    Reading,
    Materializing,
    Preparing,
    Encoding,
    Encoded,
    Submitting,
    Submitted,
    GpuComplete,
    Mapping,
    Decoding,
    StoringBlobs,
    CommittingManifest,
    RestoringPrivate,
    ReplayingPrivate,
    ComparingExpected,
    Comparing,
    Querying,
    Rebuilding,
    RestoringParticipants,
    ValidatingFinal,
    Publishing,
    CreatingGeneration,
    LoadingAnchor,
    Replaying,
    ClosingAdmission,
    Draining,
    FinalCheckpoint,
    Releasing,
}

pub struct CancelledOperation {
    pub receipt: ReceiptId,
    pub last_phase: OperationPhase,
    pub submitted_work_drained: bool,
}

pub enum CancelResult {
    CancelledBeforeSubmit,
    DeliverySuppressed,
    AbortRequested,
    NotCancellable,
    AlreadyTerminal,
}

pub struct GenesisReceipt { /* private operation handle */ }
pub struct TickReceipt { /* private operation handle */ }
pub struct InterestReceipt { /* private operation handle */ }
pub struct QueryReceipt { /* private operation handle */ }
pub struct ObservationResnapshotReceipt { /* private operation handle */ }
pub struct CheckpointReceipt { /* private operation handle */ }
pub struct CorrectionReceipt { /* private operation handle */ }
pub struct RestoreReceipt { /* private operation handle */ }
pub struct ReplayReceipt { /* private private-world replay handle */ }
pub struct RecoveryReceipt { /* private operation handle */ }
pub struct ShutdownReceipt { /* private operation handle */ }

impl GenesisReceipt {
    pub fn poll(&self) -> ReceiptState<GenesisReady, GenesisError>;
}
impl TickReceipt {
    pub fn poll(&self) -> ReceiptState<TickConfirmed, FailedNoAdvance>;
    pub fn cancel(&self) -> CancelResult;
}
impl InterestReceipt {
    pub fn poll(&self) -> ReceiptState<InterestApplied, InterestError>;
    pub fn cancel(&self) -> CancelResult;
}
impl QueryReceipt {
    pub fn poll(&self) -> ReceiptState<QueryResult, QueryUnavailable>;
    pub fn cancel(&self) -> CancelResult;
}
impl ObservationResnapshotReceipt {
    pub fn poll(&self)
        -> ReceiptState<ObservationResnapshot, ObservationSnapshotError>;
    pub fn cancel(&self) -> CancelResult;
}
impl CheckpointReceipt {
    pub fn poll(&self) -> ReceiptState<CheckpointCommitted, CheckpointError>;
    pub fn cancel(&self) -> CancelResult;
}
impl CorrectionReceipt {
    pub fn poll(&self) -> ReceiptState<CorrectionCommitted, CorrectionError>;
    pub fn cancel(&self) -> CancelResult;
}
impl RestoreReceipt {
    pub fn poll(&self) -> ReceiptState<RestoreReady, RestoreError>;
    pub fn cancel(&self) -> CancelResult;
}
impl ReplayReceipt {
    pub fn poll(&self) -> ReceiptState<ReplayCompleted, ReplayFailure>;
    pub fn cancel(&self) -> CancelResult;
}
impl RecoveryReceipt {
    pub fn poll(&self) -> ReceiptState<Recovered, RecoveryError>;
    pub fn cancel(&self) -> CancelResult;
}
impl ShutdownReceipt {
    pub fn poll(&self) -> ReceiptState<ShutdownReport, ShutdownError>;
}
```

For every non-query receipt, `OperationProgress.blocker` is `None`. A query
uses `Some(ProgressBlocker::Query(...))` only while its phase is
`WaitingForReadiness`; this makes the exact cold/materializing/revision/budget
condition observable through the same generic receipt shape.

Each concrete receipt (`GenesisReceipt`, `TickReceipt`, `InterestReceipt`,
`QueryReceipt`, `ObservationResnapshotReceipt`, `CheckpointReceipt`,
`CorrectionReceipt`, `RestoreReceipt`, `ReplayReceipt`, `RecoveryReceipt`, and
`ShutdownReceipt`) exposes `poll` specialized to its result/error types and
`cancel` where the matrix below permits it. `Ready`, `Failed`, and `Cancelled`
are terminal. An accepted operation pre-reserves one terminal-result record
and its worst-case encoded bytes. That permit remains held while any cloned
receipt handle retains the result; if the bounded cache and outstanding
handles consume all terminal permits, new result-producing admissions return
`Full`. Cloning a receipt shares one record and allocation. Moria therefore
does not grow retained results when a consumer keeps old receipts.

```text
Reserved -> Admitted -> Preparing -> Encoded -> Submitted
    -> GpuComplete -> Decoding -> Confirmed
    \-> FailedNoAdvance
```

`Confirmed` means the live root, revisions, outcome record, participant
commitments, rollback frontier, replay entry, and root hash were coordinated.
For an ordinary tick it does not mean the later asynchronous `ReplaySink`
append is durable. `Submitted` does not. TECH-047 defines the terminal-world
policy when that post-confirmation append fails: the already-terminal tick
receipt remains `Ready`, the committed frontier remains trustworthy, and a
world-lifecycle observation plus telemetry reports the persistence failure.
A correction is different: its replacement suffix is one
`CorrectionBranch` append, and `CorrectionReceipt` cannot become `Ready` or
publish its corrected frontier until that append is durable.

An admitted canonical tick cannot be consumer-cancelled. Before submission,
shutdown may explicitly abandon the whole unconfirmed tick and complete it as
`FailedNoAdvance { cause: TickNoAdvanceCause::Shutdown, ... }`; after GPU
submission Moria drains lifetime tracking and then discards the candidate
without publication. Queries,
interest materialization, presentation, and checkpoint reads may be cancelled
before encoding. After submission, cancellation suppresses delivery but does
not return resource permits until GPU/map completion. Correction, restore, and
recovery return `AbortRequested` after private submission: they suppress the
publication step, drain private resources, and retain/no-publish the live/world
state specified below. Correction has the narrower final cutoff in its matrix
row: after its branch append is invoked it returns `NotCancellable` and must
drain to durable publication or terminal provider failure. Checkpoint
`AbortRequested` stops new batches and
prevents manifest commit while submitted store/GPU calls drain.

The complete asynchronous lifecycle policy is:

| Family | Pending phases | Last cancellation point | Terminal result | Explicit retry | Device generation / shutdown |
| --- | --- | --- | --- | --- | --- |
| Genesis | `Verifying`, `Materializing`, `Submitting`, `ExportingReplayHeader` | none after `publish_genesis` is accepted | `Ready(GenesisReady)` only after replay header durability, or `Failed(GenesisError)` with no world | fix/retry the returned builder after rejection; construct a new builder after an accepted failure | loss or replay-sink failure fails construction; shutdown is unavailable before a world exists |
| Tick | states shown above | never consumer-cancellable; shutdown may abandon before publication | `Ready(TickConfirmed)` or `Failed(FailedNoAdvance)` | submit a new batch for the still-next tick only after the prior attempt is terminal | old-generation work drains and cannot publish; shutdown reports abandonment |
| Interest upsert/withdraw | `Queued`, `Applying` | before the control record is applied | `Ready(InterestApplied)` once replacement/withdrawal is installed; material readiness continues through observations | new explicit upsert/withdraw/retry | generation loss keeps the installed request but reports truth unavailable; shutdown cancels unapplied records |
| Query | `Queued`, `WaitingForReadiness`, `Encoded`, `Submitted`, `Mapping`, `Decoding` | before encoding; later cancellation suppresses delivery | `Ready(QueryResult)`, `Failed(QueryUnavailable)`, or `Cancelled` | resubmit an owned new request; never automatic | old-generation result is `DeviceLost`; shutdown cancels unsubmitted and drains submitted work |
| Observation subscription | no background receipt; `poll` reads the shared ring | `close`/drop at any time | `Items`, `Gap`, or `Closed` from `poll` | resnapshot/resume explicitly after `Gap` | generation loss is itself recorded when possible; shutdown makes the final poll `Closed` after retained items/gap |
| Observation resnapshot | `Queued`, `Pinning`, `Querying`, `Encoding` | before root/query encoding | `Ready(ObservationResnapshot)` or `Failed(ObservationSnapshotError)` | new bounded request, including after an immediate resume gap | old generation fails; shutdown cancellation follows query rules |
| Checkpoint | `Queued`, `Pinning`, `Reading`, `StoringBlobs`, `CommittingManifest` | before first GPU readback/store call; later cancellation stops new batches and drains submitted calls | `Ready(CheckpointCommitted)`, `Failed(CheckpointError)`, or `Cancelled` with no manifest | new explicit request against a retained frontier | device loss/store failure leaves no committed manifest; shutdown either completes the configured required request or reports it failed |
| Correction | `Queued`, `RestoringPrivate`, `ReplayingPrivate`, `ValidatingFinal`, `ExportingCorrectionBranch`, `Publishing` | cancellation before the correction-branch sink invocation aborts and drains private state; after invocation it is `NotCancellable` because durable success mandates publication | `Ready(CorrectionCommitted)` only after the one branch append is durable and the corrected bundle/log swap completes; every `Failed(CorrectionError)` reports the unchanged `original_frontier` and `CommittedEffect::None`; branch-append failure additionally applies TECH-047's terminal-world provider policy | new complete correction request while the world remains `Ready` | old-generation private results cannot reach branch export; shutdown aborts before invocation, but after invocation it drains to durable publication or terminal provider failure |
| Restore | `Loading`, `Verifying`, `Rebuilding`, `RestoringParticipants`, `ExportingReplayHeader`, `Publishing` | before device/store submission; after either device/store submission or replay-header invocation, cancellation drains the private builder and suppresses publication | `Ready(RestoreReady)` only after the checkpoint-anchored sequence-zero replay header is durable, or `Failed(RestoreError)` with no world published | retry with a new builder/request and a different stream after an invoked header; a pre-invocation failure releases its reservation | generation loss or replay-sink failure fails private construction; no world exists for shutdown |
| Public replay | `LoadingOwnedRecords`, `VerifyingHeader`, `ReplayingPrivate`, `ComparingExpected`, `ExportingReplayHeader`, `ExportingReplayPrefix`, `Publishing` | before first private submission; later cancellation drains the private builder, any invoked sink calls, and suppresses publication | `Ready(ReplayCompleted)` only after the verified source header/records have been copied durably to the selected fresh live stream, or `Failed(ReplayFailure)` with no world published; divergence includes the bounded artifact | new builder/owned replay request and a different stream after any sink invocation | old-generation results cannot install; sink failure fails construction; no world exists for shutdown until final publication |
| Recovery | `Queued`, `CreatingGeneration`, `LoadingAnchor`, `Replaying`, `Comparing` | before new-generation submission; later cancellation remains in `RecoveringParticipant` | `Ready(Recovered)` or `Failed(RecoveryError)` | one explicit new recovery request | only results from the requested new generation may reinstall the equal frontier; shutdown abandons recovery |
| Shutdown | `ClosingAdmission`, `Draining`, `FinalCheckpoint`, `Releasing` | not cancellable | `Ready(ShutdownReport)` or `Failed(ShutdownError)` after all safe release work | none; world remains `Closed` | old-generation completions are lifetime acknowledgements only |

Environmental retry never reuses a consumed request or receipt. A retry is a
new admission with a new reservation and receipt, except an observation
subscription's in-place `resume`, which merely changes its cursor after
validation and allocates no historical record. No operation has a timer-based
retry loop. Terminal errors always state whether a frontier changed; only
`TickConfirmed`, `CorrectionCommitted`, and successful genesis/restore/recovery
publication can change the canonical frontier.

Materialization, participant preparation/export, presentation derivation, and
checkpoint blob/store calls are subordinate bounded jobs, not additional
facade operation families. Their admission reserves under the owning
interest/query/tick/checkpoint receipt before invoking a producer.
Materialization cancellation before GPU encoding returns its permit; after
submission it drains, with readiness reported by lifecycle observations or the
waiting query. Participant jobs follow TECH-016/029 and terminate their owning
genesis/tick/checkpoint/correction/restore/replay/recovery receipt. Presentation work
is admitted only for installed interest/current dirty revisions, coalesces
before submission, drains/discards after withdrawal or staleness, and reports
`Current`/`Failed` through TECH-056 observations rather than a second receipt.
Checkpoint store jobs terminate the checkpoint/shutdown receipt. In every
case old-generation output can release resources but cannot install, and retry
requires a new owning request or explicit lifecycle retry.

## Interest and lifecycle

### TECH-022 — Bounded interest

Implements: REQ-004, REQ-009, REQ-016, REQ-018, REQ-031

```rust
pub struct InterestId(u64);

pub enum VolumeSelector {
    One(VolumeId),
    Listed(BoundedVec<VolumeId>),
}

pub struct InterestRequest {
    pub id: InterestId,
    pub world: WorldId,
    pub volumes: VolumeSelector,
    pub bounds: WorldAabbQ,
    pub capabilities: InterestCapabilities,
    pub priority: u8,
    pub max_resident_bricks: u32,
    pub allow_partial: bool,
}

pub struct InterestCapabilities {
    pub inspect: bool,
    pub collision: bool,
    pub presentation: bool,
    pub preload_for_activation: bool,
}

pub struct InterestWithdrawal {
    pub world: WorldId,
    pub id: InterestId,
}

pub struct InterestCapacity {
    pub bricks: u64,
}
```

`InterestId` has infallible `from_raw(u64)` and `get(self) -> u64`; all bit
patterns are valid and uniqueness among live interests is checked at
admission.

The TECH-070 `upsert_interest` and `withdraw_interest` calls use a bounded
noncanonical control queue and return `InterestReceipt`; admission rejection
returns the complete request. Interest IDs are consumer-owned and replace
prior requests atomically. Moria clips no request silently: the consumer
either sets `allow_partial` and receives the exact covered bounds, or receives
`AdmissionError { code: InterestTooLarge, context:
AdmissionContext::InterestCapacity { required, supported }, ... }`. Both
capacity values are exact brick counts calculated with checked arithmetic;
`supported` is the lesser of `max_resident_bricks` and the applicable world
budget. At least one capability must be true. `VolumeSelector` is either one
explicit volume or an owned bounded, sorted, unique list; there is no implicit
future-volume selector.

Per `(volume, brick-range, capability)` lifecycle follows:

```text
Cold -> Requested -> Materializing -> Ready -> Retiring -> Cold
                           \-> Failed -> Requested (explicit retry)
```

Withdrawal makes state eligible for `Retiring`; pins, unpersisted scars,
admitted work, snapshots, checkpoints, or observation recovery delay eviction.
Lifecycle is noncanonical and cannot change simulation-domain membership.
`PreloadForActivation` verifies and pins exact content but only
`ActivateRegion` in a tick changes canonical membership.

## Query and collision facade

### TECH-023 — Query contract

Implements: REQ-005, REQ-010, REQ-017, REQ-019, REQ-021

```rust
pub struct QueryRequest {
    pub world: WorldId,
    pub at: QueryFrontier,
    pub freshness: QueryFreshness,
    pub scope: QueryScope,
    pub kind: QueryKind,
    pub completeness: Completeness,
    pub limits: QueryLimits,
}

pub enum QueryFrontier {
    LatestCommitted,
    Retained { tick: Tick, root_hash: CanonicalHash },
}

pub struct QueryFreshness {
    pub minimum: BoundedVec<MinimumVolumeRevision>,
    pub if_unmet: MinimumRevisionPolicy,
}

pub struct MinimumVolumeRevision {
    pub volume: VolumeId,
    pub revision: VolumeRevision,
}

pub enum MinimumRevisionPolicy {
    Wait,
    ReturnStale,
}

pub enum QueryKind {
    Sample { point: WorldPointQ },
    Region { bounds: WorldAabbQ },
    Occupancy { shape: CollisionShapeQ },
    Trace { segment: SegmentQ, max_hits: u32 },
    Overlap { shape: CollisionShapeQ, max_hits: u32 },
    Sweep { shape: CollisionShapeQ, delta: WorldVectorQ, max_hits: u32 },
}

pub enum QueryScope {
    World { volumes: BoundedVec<VolumeId> },
    Volume { volume: VolumeId, local_bounds: LocalCellAabb },
}

pub enum Completeness {
    Complete,
    ExplicitPartial,
}

pub struct QueryLimits {
    pub max_bricks: u32,
    pub max_records: u32,
    pub max_result_bytes: u64,
    pub max_workgroups: u32,
    pub max_volume_revisions: u32,
}

pub struct QueryCapacity {
    pub bricks: u64,
    pub records: u64,
    pub result_bytes: u64,
    pub workgroups: u64,
    pub volume_revisions: u64,
}

pub struct MinimumRevisionGap {
    pub volume: VolumeId,
    pub required: VolumeRevision,
    pub current: Option<VolumeRevision>,
}

pub enum QueryReadinessReason {
    Availability {
        missing: BoundedVec<MissingRange>,
    },
    MinimumRevision {
        unmet: BoundedVec<MinimumRevisionGap>,
    },
    ResourcePressure {
        field: ResourceBudgetField,
        required: u64,
        supported: u64,
    },
}
```

Query limits cap inspected bricks, returned cells/hits, encoded result bytes,
and workgroups. A request may lower but not exceed the corresponding
`ResourceBudgets.query` field; the configured defaults are 4,096 bricks,
65,536 records, 4 MiB result bytes, and 256 volume revisions, with portable
maxima 16,384, 262,144, 16 MiB, and 1,024 respectively. Workgroups are checked
against both the derived brick/record count and the granted device limit. A
request larger than either its own or world limit is rejected before work and
returned intact.
`minimum` is limited by `QueryLimits.max_volume_revisions`, must contain unique
`VolumeId`s in increasing order after admission normalization, and every ID
must be selected by the query scope.
Before admission, Moria derives a conservative `QueryCapacity` from the
request scope/kind and compares every dimension with a second `QueryCapacity`
formed from the lesser of request and world/device limits. Any exceeded
dimension rejects with the two complete records in
`AdmissionContext::QueryCapacity`; unused dimensions are zero, so the
consumer never has to infer which bound failed from a fieldless code.

`Completeness::Complete` returns
`ReceiptState::Pending(OperationProgress { phase:
WaitingForReadiness, blocker: Some(ProgressBlocker::Query(reason)) })` or a
typed terminal `QueryUnavailable`, rather than partial data.
`QueryReadinessReason` identifies the exact cold/materializing ranges,
unmet revision floors, or bounded resource pressure. `ExplicitPartial`
returns exact inspected bounds plus missing subranges and never describes
them as empty. Every ready result carries the closed frontier position, world
root hash, sorted
per-volume revisions, exact inspected bounds, completeness, and source
commitment.

Ordinary results are noncanonical observations. A participant may affect a
later tick only by encoding the result or its commitment into a later
`ParticipantInput`. Tick-synchronous collision uses the participant artifact
contract rather than callback arrival.

### TECH-024 — Query result ordering and freshness

Implements: REQ-010, REQ-012, REQ-017, REQ-019

Matter samples are ordered by `(volume_id, local z, local y, local x)`.
Collision encounters are ordered by `(time_of_impact, volume_id, local z,
local y, local x, face_id)`. Equal fixed-point times retain the remaining key
order. Output capacity is precomputed. An oversized request is rejected with
`AdmissionCode::ResultCapacityExceeded` and
`AdmissionContext::QueryCapacity`; a larger-than-proven result discovered
after admission terminates as
`QueryUnavailable::ResultCapacityExceeded { required, supported }`. Both
paths use exact `QueryCapacity` records and no complete result is silently
truncated.

A query pins its root at admission. It may complete after later ticks and
still truthfully reports its older frontier. A `freshness.minimum` entry that
is not met returns pending or stale according to the request; Moria does not relabel.
Results from a reclaimed, device-lost, or hash-mismatched root finish
`Unavailable`, not empty. For `LatestCommitted`, `Wait` leaves the request
pending without pinning an older root until every minimum is met;
`ReturnStale` pins the current root and returns a result explicitly labeled
`stale` with each unmet pair. A retained frontier can never become newer, so
an unmet `Wait` condition returns `Unavailable(FrontierTooOld)` and
`ReturnStale` behaves as above.

## Observations and telemetry

### TECH-025 — Gap-aware observation stream

Implements: REQ-005, REQ-012, REQ-017, REQ-021

```rust
pub struct CorrelationId([u8; 16]);
pub struct ObservationStreamId([u8; 16]);

pub struct ObservationSubscriptionRequest {
    pub world: WorldId,
    pub volumes: BoundedVec<VolumeId>,
    pub kinds: ObservationKindFilter,
    pub spatial: Option<WorldAabbQ>,
    pub include_world_events: bool,
    pub start: ObservationStart,
}

pub struct ObservationKindFilter {
    pub canonical_outcomes: bool,
    pub volume_lifecycle: bool,
    pub material_region_lifecycle: bool,
    pub presentation: bool,
    pub checkpoint: bool,
    pub correction: bool,
    pub device_and_world_lifecycle: bool,
}

pub enum ObservationStart {
    Now,
    OldestRetained,
    After(ObservationCursor),
}

pub struct ObservationSubscription { /* private stream/cursor handle */ }

pub struct ObservationCursor {
    pub stream: ObservationStreamId,
    pub next_sequence: u64,
}

pub enum ObservationCursorError {
    WrongStream,
    BeforeLastTrustworthy,
    BeyondTail,
    NotProducedForSubscription,
    Closed,
}

pub struct ObservationPollLimits {
    pub max_records: u32,
    pub max_bytes: u64,
}

pub enum ObservationPoll {
    Items {
        next: ObservationCursor,
        items: BoundedVec<Observation>,
    },
    Gap {
        last_trustworthy: ObservationCursor,
        oldest_available: ObservationCursor,
        resnapshot_at: FrontierSummary,
        correlation_lost: bool,
    },
    Closed,
}

pub struct ObservationResnapshotRequest {
    pub at: FrontierSummary,
    pub query: Option<QueryRequest>,
    pub max_volume_summaries: u32,
    pub max_region_summaries: u32,
    pub max_bytes: u64,
}

pub struct ObservationResnapshot {
    pub frontier: FrontierSummary,
    pub volumes: BoundedVec<ObservedVolumeSummary>,
    pub regions: BoundedVec<ObservedRegionSummary>,
    pub query: Option<QueryResult>,
    pub resume_at: ObservationCursor,
}

pub enum ObservationCloseResult {
    Closed,
    AlreadyClosed,
}

pub struct Observation {
    pub sequence: u64,
    pub frontier: Option<FrontierSummary>,
    pub order: Option<CanonicalOrder>,
    pub kind: ObservationKind,
    pub volumes: BoundedVec<VolumeId>,
    pub world_bounds: Option<WorldAabbQ>,
    pub correlation: Option<CorrelationMetadata>,
}

pub enum ObservationKind {
    CanonicalOutcome(CommandOutcome),
    VolumeLifecycle(VolumeLifecycleFact),
    MaterialRegionLifecycle(RegionLifecycleFact),
    Presentation(PresentationFact),
    Checkpoint(CheckpointObservation),
    Correction(CorrectionObservation),
    DeviceAndWorld(WorldLifecycleFact),
}

pub struct ObservedVolumeSummary {
    pub volume: VolumeId,
    pub revision: VolumeRevision,
    pub kind: VolumeKind,
    pub placement: PlacementQ,
    pub retired: bool,
}

pub struct ObservedRegionSummary {
    pub volume: VolumeId,
    pub local: LocalCellAabb,
    pub state: MaterialRegionState,
    pub revision: VolumeRevision,
}

pub enum MaterialRegionState {
    Cold,
    Requested,
    Materializing,
    Ready,
    Retiring,
    Failed,
}

pub struct VolumeLifecycleFact {
    pub volume: VolumeId,
    pub revision: VolumeRevision,
    pub change: VolumeLifecycleChange,
}

pub enum VolumeLifecycleChange {
    Created,
    Moved { old: PlacementQ, new: PlacementQ },
    Retired,
}

pub struct RegionLifecycleFact {
    pub volume: VolumeId,
    pub local: LocalCellAabb,
    pub state: MaterialRegionState,
    pub revision: VolumeRevision,
    pub failure: Option<ErrorCode>,
}

pub struct PresentationFact {
    pub volume: VolumeId,
    pub local: LocalCellAabb,
    pub source_revision: VolumeRevision,
    pub status: PresentationStatus,
}

pub enum PresentationStatus {
    Absent,
    Building,
    Current,
    Stale,
    Failed,
}

pub struct CheckpointObservation {
    pub key: CheckpointKey,
    pub store: CheckpointStoreId,
    pub durable: Option<DurableFrontier>,
    pub failure: Option<ErrorCode>,
}

pub struct CorrectionObservation {
    pub from: FrontierSummary,
    pub to: Option<FrontierSummary>,
    pub replay: Option<ReplayStreamPosition>,
    pub failure: Option<ErrorCode>,
}

pub struct WorldLifecycleFact {
    pub state: WorldState,
    pub generation: DeviceGeneration,
    pub frontier: FrontierSummary,
    pub failure: Option<OperationError>,
    pub replay_export_failure: Option<ReplayExportFailure>,
}
```

`WorldLifecycleFact.frontier` is the live, last trustworthy frontier at the
instant the fact is appended; it is never a candidate that failed before
publication. Consequently a correction-branch append failure reports the
original frontier here, while an ordinary post-confirmation tick append
failure reports that already-confirmed tick frontier. When `failure` is
present, its `CommittedEffect` describes the failed causal operation rather
than duplicating this snapshot field.

`CorrelationId` has infallible `from_bytes`, `as_bytes`, and `to_bytes`
methods; all bit patterns are consumer-valid. `ObservationStreamId` is
Moria-created and exposes `as_bytes` and `to_bytes` but no public constructor.

Subscription admission validates a nonempty kind filter and resolves
`volumes` to one sorted, unique, finite set of IDs already present in the
world registry. Its length cannot exceed
`observation.volumes_per_subscription`; an empty set means no volume-scoped
records, not all volumes. Future volume creation does not enlarge membership.
`include_world_events` is the only way to receive records without a volume
identity, such as checkpoint/world/device lifecycle. A spatial filter applies
only to records with spatial evidence; world events do not match it.
`VolumeCreated` is delivered as a world-discovery event when
`include_world_events` is true,
even though its append facts carry the new ID and placed bounds; later records
for that volume require a newly admitted subscription that includes it. No
other volume-scoped record bypasses fixed membership. An
`After` cursor must belong to this world's current stream and may immediately
produce `Gap`. Rejection returns the complete request.

Each world has one count-and-byte bounded ring, default 8,192 records and
32 MiB payload, configurable only at genesis. Every record reserves and stores
its full encoded payload plus immutable append-time `ObservationFilterFacts`:

```text
kind tag;
sorted affected VolumeId list;
optional committed old and new volume revision;
optional world-space affected AABB at the record's committed placement;
for movement, the union of old and new placed volume bounds;
for retirement, the last committed placed bounds;
for creation, the first committed placed bounds;
canonical tick/order/root and directory-version digest at append.
```

The stream ID is assigned at genesis, survives correction and device recovery,
and closes at shutdown. Durable restore constructs a new world/stream even
when it restores the same canonical `WorldId`, so an old cursor cannot be
accepted accidentally.

Material-change bounds are transformed using that tick's placement before the
record is appended. Lifecycle and presentation records use the source
revision/placement they report. Polling applies membership, kind, and spatial
intersection only to these stored facts; it never asks the current placement,
live volume registry, or a reclaimed directory version what was historically
true. The directory-version digest is evidence, not a pin. A record whose
facts exceed `bytes_per_record` is replaced by an explicit
`ObservationRecordTooLarge` gap marker covering its sequence; it is never
silently omitted.

Records have noncanonical stream sequence plus canonical tick, within-tick
order, root hash, relevant revisions, and contract version. Coalescing is
allowed only for lifecycle/presentation telemetry, only when the complete
append-time filter facts and final state remain representable, and retains the
covered sequence range. Canonical outcome observations are never coalesced.

An outcome observation also carries the optional bounded correlation sidecar
from TECH-020. Correlation expiry follows ring expiry and is never synthesized
after a gap; all canonical fields remain independently usable.

`poll` returns no more than both request and genesis poll limits. It advances
over nonmatching records as well as returned records, but its `next` cursor
always proves the exact scanned sequence; it cannot conceal an overwritten
range. Count or byte overwrite advances `oldest_available` and produces
`Gap`; no API returns the newest cursor while hiding lost history. Closing is
idempotent, releases the subscription record, and makes all future polls
`Closed`. Dropping the subscription performs the same close. It does not
cancel an already accepted resnapshot receipt.

During `ShuttingDown`, the stream records one final world-lifecycle item when
capacity permits, freezes its tail, and permits polls against retained
items/gaps while lifetime draining proceeds. Shutdown does not wait for
consumers to poll; when `ShutdownReceipt` becomes terminal, all still-open
subscriptions return `Closed` and release their records.

Gap recovery is the TECH-070 resnapshot call, not an unbounded event replay.
Admission verifies that the requested frontier is retained, every query volume
is in the subscription's fixed membership, the query bounds are within its
spatial filter when present, and all count/byte limits fit both the request and
`ResourceBudgets.observation`. One root and the ring's current next cursor are
pinned atomically. The result contains current volume revision/placement/
retirement summaries, requested material-region lifecycle summaries, and the
optional ordinary bounded query at that same root; it never enumerates more
than the admitted finite membership or reports omitted state as empty.
`resume_at` is the cursor captured with that root. After `Ready`, the consumer
calls `resume(resume_at)` and polls newer records. If the ring overwrote that
cursor while the snapshot ran, the first poll returns another explicit gap.
`resume` accepts only this subscription's last successful poll cursor or an
unconsumed `resume_at` produced for it, never a cursor beyond the current ring
tail or from another stream; moving backward before the last trustworthy
cursor is rejected. Failed/cancelled snapshots leave the existing subscription
cursor unchanged. Delivery order and resnapshot timing cannot affect ticks.

### TECH-026 — Telemetry without storage exposure

Implements: REQ-004, REQ-018, REQ-022

`TelemetrySnapshot` reports configuration fingerprint, authority status, and
adapter diagnostic context; world/region/presentation states; logical and
physical residency; queue/pool current, capacity, and high-water marks; oldest
operation age;
tick/frontier/replay depth; changed leaves/hash nodes; observation gaps;
checkpoint coverage; participant transfer/readback bytes; failures; and
timings/histograms named in the design.

It reports consumer-meaningful counts and bytes, never buffer handles, device
addresses, physical voxel slots, mutable ECS entities, or an iterator over
internal storage. Telemetry is bounded, sampled after the fact, and explicitly
noncanonical. Canonical hashes and outcomes are copied into it as labels; the
telemetry system does not calculate them.

## Error and world lifecycle

### TECH-027 — Typed failure taxonomy

Implements: REQ-005, REQ-015, REQ-021

Public failures retain actionable layers:

```rust
pub enum ErrorCode {
    InvalidConfig,
    DuplicateId,
    MissingId,
    WrongProviderKind,
    ContractMismatch,
    UnsupportedVersion,
    InvalidBounds,
    InvalidEncoding,
    InvalidOrientation,
    InvalidCell,
    WrongWorld,
    WorldUnknown,
    BeforeNextTick,
    AfterNextTick,
    AlreadyPending,
    WorldNotReady,
    WorldClosed,
    WorldFailed,
    TelemetryBusy,
    AlreadyShuttingDown,
    DependencyNotReady,
    QueueFull,
    CapacityExceeded,
    CanonicalBudget,
    PersistenceBackpressure,
    StaleRevision,
    StaleHash,
    ArithmeticOverflow,
    SourceUnavailable,
    SourceInvalid,
    ProducerDropped,
    StoreFailure,
    ManifestNotFound,
    UnsupportedAtomicCommit,
    CorruptBlob,
    LineageMismatch,
    FrontierUnavailable,
    FrontierTooOld,
    ResultCapacityExceeded,
    ObservationGap,
    ParticipantFailure,
    ParticipantDivergence,
    ReplayDivergence,
    BackendUnavailable,
    DeterminismViolation,
    DeviceLost,
    MappingFailure,
    DecodeFailure,
    Cancelled,
    Shutdown,
    InternalInvariant,
}

pub enum FailureScope {
    Configuration,
    World(WorldId),
    Tick { world: WorldId, tick: Tick },
    Volume { world: WorldId, volume: VolumeId },
    Operation(ReceiptId),
    Provider(ProviderId),
}

pub enum ProviderId {
    InputSource(InputSourceId),
    BaseSource(BaseContentSourceId),
    BaseAuthority(BaseAuthorityId),
    ContentBlobStore(ContentBlobStoreId),
    CheckpointStore(CheckpointStoreId),
    ReplaySink(ReplaySinkId),
    Participant(ParticipantId),
}

pub enum BudgetGroup {
    Identity,
    Canonical,
    Content,
    Query,
    Observation,
    Presentation,
    Checkpoint,
    Rollback,
    Participant,
    Runtime,
}

pub struct ResourceBudgetField {
    pub group: BudgetGroup,
    pub field_code: u16,
}

pub enum Retryability {
    RetryNewRequest,
    RetryAfterDependency,
    RetryAfterRecovery,
    Never,
}

pub enum CommittedEffect {
    None,
    Frontier(FrontierSummary),
}

pub struct OperationError {
    pub code: ErrorCode,
    pub scope: FailureScope,
    pub retryability: Retryability,
    pub committed: CommittedEffect,
    pub diagnostic: BoundedUtf8<160>,
}

pub struct CorrectionError {
    pub original_frontier: FrontierSummary,
    pub error: OperationError,
    pub replay_export_failure: Option<ReplayExportFailure>,
}

pub enum TickNoAdvanceCause {
    Canonical(CanonicalFailure),
    Participant {
        participant: ParticipantId,
        code: ErrorCode,
    },
    Provider {
        provider: ProviderId,
        code: ErrorCode,
    },
    Device {
        generation: DeviceGeneration,
        code: ErrorCode,
    },
    Shutdown,
    Internal(ErrorCode),
}

pub struct FailedNoAdvance {
    pub world: WorldId,
    pub attempted_tick: Tick,
    pub source_frontier: FrontierSummary,
    pub cause: TickNoAdvanceCause,
    pub error: OperationError,
}

pub enum TelemetryError {
    WorldUnknown {
        world: WorldId,
    },
    WorldClosed {
        world: WorldId,
        last_frontier: Option<FrontierSummary>,
    },
    TelemetryBusy {
        world: WorldId,
    },
}

pub struct ConfigError {
    pub code: ConfigErrorCode,
    pub field: ConfigField,
    pub diagnostic: BoundedUtf8<160>,
}

pub enum ConfigErrorCode {
    DuplicateId,
    RetiredReplayStreamCapacity,
    MissingReference,
    WrongProviderKind,
    ContractMismatch,
    InvalidValue,
    CrossLimitViolation,
    UnsupportedCapability,
    ArithmeticOverflow,
}

pub enum ConfigField {
    Canonical,
    Budgets(ResourceBudgetField),
    Rollback,
    Persistence,
    Presentation,
    Execution,
    Material,
    InputSource,
    BaseSource,
    BaseAuthority,
    ContentBlobStore,
    CheckpointStore,
    ReplaySink,
    Volume,
    Participant,
}

pub struct AdmissionError {
    pub code: AdmissionCode,
    pub retryability: Retryability,
    pub context: AdmissionContext,
}

pub enum AdmissionContext {
    None,
    TickEligibility {
        supplied: Tick,
        expected_next: Tick,
    },
    InvalidBatch {
        reason: BatchError,
    },
    InterestCapacity {
        required: InterestCapacity,
        supported: InterestCapacity,
    },
    QueryCapacity {
        required: QueryCapacity,
        supported: QueryCapacity,
    },
    CorrectionExpectedHashCount {
        replacement_batches: u32,
        expected_hashes: u32,
    },
    BudgetCapacity {
        field: ResourceBudgetField,
        required: u64,
        supported: u64,
    },
}

pub enum AdmissionCode {
    WrongWorld,
    WrongState,
    DuplicateReplayStream,
    RetiredReplayStreamCapacity,
    BeforeNextTick,
    AfterNextTick,
    AlreadyPending,
    WorldNotReady,
    DependencyNotReady,
    Full,
    Closed,
    InvalidRequest,
    InvalidBatch,
    InterestTooLarge,
    ResultCapacityExceeded,
    CorrectionHashCountMismatch,
    StaleGeneration,
    PersistenceBackpressure,
}

pub struct ReserveError {
    pub code: AdmissionCode,
    pub supported: TickReservation,
}

pub enum PushError {
    LimitExceeded,
    DuplicateSourceSequence,
    UnregisteredSource,
    InvalidInput,
}

pub enum BatchError {
    Empty,
    CountMismatch,
    DuplicateCanonicalKey,
    EncodingFailure,
    ReservationMismatch,
}

pub enum CanonicalFailure {
    MissingIdentity,
    WrongVolumeKind,
    StaleRevision,
    StaleSourceHash,
    InvalidBounds,
    InvalidCell,
    InvalidOrientation,
    InvalidFixedFormat,
    ArithmeticOverflow,
    DivisionByZero,
    InvalidShift,
    NegativeSquareRoot,
    Nonrepresentable,
    LogicalCapacity,
    DependencyUnavailable,
    ParticipantEffectInvalid,
    ParticipantFailed,
    InjectedCandidateFailure,
    ZeroAxis,
    UnrepresentableAxis,
}

pub enum AvailabilityCode {
    Cold,
    Materializing,
    Failed,
    FrontierTooOld,
    DeviceLost,
    CapacityExceeded,
}

pub enum MoriaError {
    Config(ConfigError),
    Admission(AdmissionError),
    Operation(OperationError),
    Tick(FailedNoAdvance),
    Query(QueryUnavailable),
    Telemetry(TelemetryError),
}

pub enum QueryUnavailable {
    Availability {
        error: OperationError,
        missing: BoundedVec<MissingRange>,
    },
    ResultCapacityExceeded {
        required: QueryCapacity,
        supported: QueryCapacity,
    },
}

pub type GenesisError = OperationError;
pub type InterestError = OperationError;
pub type ObservationSnapshotError = OperationError;
pub type CheckpointError = OperationError;
pub type RestoreError = OperationError;
pub type ParticipantError = OperationError;
pub type RecoveryError = OperationError;
pub type ShutdownError = OperationError;
```

`ZeroAxis` and `UnrepresentableAxis` are appended stable wire tags; every
existing `CanonicalFailure` variant keeps its prior tag. TECH-007/071 are the
only producers of these two axis-specific failures.

`ResourceBudgetField.field_code` is the stable closed ordinal of the named
TECH-017 field within its `BudgetGroup`; decoding rejects any ordinal not
present in that version. It does not accept a string extension. Operation
aliases intentionally use one closed shape so every applicable receipt exposes
the same scope/retry/committed-effect contract while preserving the stable
`ErrorCode`. `FailedNoAdvance`, `CorrectionError`, `TelemetryError`, and
`QueryUnavailable` are the deliberate specialized shapes: tick-global failure
must retain the attempted tick and structured cause; correction failure must
retain the original still-live frontier and optional prepublication
branch-export failure; synchronous telemetry has no operation receipt; and
REQ-010/REQ-021 require exact unavailable ranges and supported result bounds.
`QueryUnavailable::Availability.error` supplies the same
scope/retry/committed fields, and
its capacity variant is a terminal no-frontier-change outcome with
`RetryNewRequest`. `AdmissionContext` must match its code: tick context is
mandatory only for `BeforeNextTick`/`AfterNextTick`, batch context only for
`InvalidBatch`, interest/query capacity context only for their corresponding
codes, `CorrectionExpectedHashCount` is mandatory only for
`CorrectionHashCountMismatch`, and `BudgetCapacity` is required for
`RetiredReplayStreamCapacity`; every other code requires `None`. A mismatched
code/context pair is an internal construction error and is never emitted.
Every `FailedNoAdvance` has
`source_frontier.next_tick() == Some(attempted_tick)`,
`error.scope == FailureScope::Tick { world, tick: attempted_tick }`, and
`error.committed == CommittedEffect::None`. Its cause retains a participant or
provider ID whenever that actor failed; device loss retains the attempted
generation. `cause` and `error.code` must describe the same stable class.
`NoAdvanceExplicitRetry` uses `RetryNewRequest` or
`RetryAfterDependency/Recovery` as applicable; a failure that transitions the
world to `Failed` uses `Never`. No tick-global failure is represented only by
a diagnostic string.

Every `CorrectionError` has
`error.committed == CommittedEffect::None` and
`original_frontier` byte-equal to the live frontier that remained installed.
`replay_export_failure` is `Some` only when the one correction-branch append
failed after invocation; in that case `error.code == StoreFailure`,
`error.scope == Provider(ReplaySink(replay_export_failure.sink))`,
`error.retryability == Never`, and the world transitions to `Failed`.
Pre-invocation validation, participant, resource, cancellation, and
divergence failures carry `None` and leave the world in the state selected by
their ordinary failure policy. This is distinct from an ordinary tick's
post-confirmation replay append failure: that lifecycle error carries
`CommittedEffect::Frontier(the_confirmed_tick_frontier)` because the causal
tick already published. `CommittedEffect::None` never means that the world has
no prior frontier; `WorldLifecycleFact.frontier` and, for corrections,
`CorrectionError.original_frontier` report that already-existing state.

`TelemetryError::WorldUnknown` and `WorldClosed` are nonretryable and have no
committed effect; the latter optionally reports the last already-copied
frontier. `TelemetryBusy` is retryable with a new synchronous call, reports no
snapshot, and cannot drive progress. These variants allocate no diagnostic and
are pattern-matchable independently of `OperationError`.

Internal errors and uncaptured GPU validation errors never panic a consumer
process; they fail the affected candidate/world and preserve the last
trustworthy frontier. Decode, corruption, unsupported version, lineage
mismatch, device loss, unsupported capability, and canonical determinism
violations remain distinct.

Canonical failures have stable wire tags. Environmental failures do not enter
canonical hashes and cannot turn into a timing-dependent canonical outcome.

### TECH-028 — World lifecycle and shutdown

Implements: REQ-008, REQ-015, REQ-016, REQ-021

```rust
pub struct ShutdownRequest {
    pub world: WorldId,
    pub persistence: ShutdownPersistence,
}

pub enum ShutdownPersistence {
    RequireCheckpoint(CheckpointRequest),
    ReportDirtyWithoutCheckpoint,
}

pub enum WorldState {
    Configuring,
    VerifyingGenesis,
    Ready,
    Replaying,
    RecoveringParticipant,
    Failed,
    ShuttingDown,
    Closed,
}
```

```text
Configuring -> VerifyingGenesis -> Ready <-> Replaying
                              \-> Failed
Ready/Replaying -> RecoveringParticipant -> Ready | Failed
Ready/Replaying/RecoveringParticipant/Failed -> ShuttingDown -> Closed
```

`Ready -> Failed` includes TECH-047's failure to durably append an
already-confirmed replay record. That transition does not revoke the confirmed
frontier or rewrite its terminal tick receipt; it closes new authority
admission and reports the committed persistence failure through the world
lifecycle observation and telemetry before shutdown.

`Failed` exposes the last trustworthy frontier but rejects new ticks; bounded
queries against pinned readable roots may continue when their failure scope
allows. Shutdown:

1. closes permits and rejects new ticks, queries, interests, and checkpoints;
2. abandons any unsubmitted unconfirmed tick with explicit receipts;
3. drains submitted GPU work for lifetime safety without publishing it;
4. completes or explicitly fails the configured required checkpoint;
5. completes/gaps observation subscribers and participant leases;
6. releases derived resources, then canonical roots, then the device adapter.

Shutdown never creates a tick. Dirty scars remain reported as nondurable if the
checkpoint fails. `ShutdownReceipt` returns the last confirmed/durable
frontiers and every abandoned receipt ID. `ReportDirtyWithoutCheckpoint` is an
explicit caller choice and returns the complete dirty-root summary; it never
reports those roots durable. `RequireCheckpoint` reserves and runs that exact
bounded checkpoint after admission closes. A second shutdown call is rejected
as `AlreadyShuttingDown` or reports `WorldClosed`; it does not create another
drain operation.

## Deterministic participant API

### TECH-029 — Runtime-neutral participant adapter

Implements: REQ-005, REQ-006, REQ-017, REQ-030, REQ-033, REQ-035

```rust
pub trait CanonicalParticipant: Send + Sync + 'static {
    fn descriptor(&self) -> ParticipantDescriptor;
    fn prepare_genesis(
        &self,
        request: ParticipantGenesisRequest,
        sink: ParticipantStateSink,
    );
    fn prepare_tick(
        &self,
        source: ParticipantStateLease,
        request: ParticipantTickRequest,
        sink: ParticipantCompletionSink,
    );
    fn restore_snapshot(
        &self,
        request: ParticipantSnapshotRestoreRequest,
        snapshot: OwnedBytes,
        sink: ParticipantStateSink,
    );
    fn reconstruct(
        &self,
        request: ParticipantReconstructRequest,
        log: ParticipantReplayLease,
        sink: ParticipantStateSink,
    );
    fn export_snapshot(
        &self,
        source: ParticipantStateLease,
        request: ParticipantSnapshotExportRequest,
        sink: ParticipantSnapshotSink,
    );
}

pub struct ParticipantGenesisRequest {
    pub world: WorldId,
    pub genesis_digest: CanonicalHash,
}
pub struct ParticipantTickRequest {
    pub world: WorldId,
    pub tick: Tick,
    pub source_frontier: FrontierPosition,
    pub source_root: CanonicalHash,
    pub input: BoundedBytes,
    pub collider: ColliderArtifactLease,
}
pub struct ParticipantSnapshotRestoreRequest {
    pub world: WorldId,
    pub frontier: FrontierSummary,
    pub expected_commitment: CanonicalHash,
}
pub struct ParticipantReconstructRequest {
    pub world: WorldId,
    pub start: FrontierSummary,
    pub end_tick: Tick,
    pub expected_commitment: CanonicalHash,
}
pub struct ParticipantSnapshotExportRequest {
    pub world: WorldId,
    pub frontier: FrontierSummary,
    pub max_bytes: u64,
}

pub struct ParticipantStateLease { /* private immutable token lease */ }
pub struct ParticipantReplayLease { /* private bounded exact record lease */ }
pub struct ColliderArtifactLease { /* private source-hash-bound artifact lease */ }
pub struct ParticipantTokenMetadata {
    pub participant: ParticipantId,
    pub contract: ContractDigest,
    pub frontier: FrontierSummary,
    pub commitment: CanonicalHash,
    pub state_bytes: u64,
    pub device_generation: Option<DeviceGeneration>,
}
pub struct ParticipantReplayRecordView<'a> {
    pub tick: Tick,
    pub digest: BlobDigest,
    pub bytes: &'a [u8],
}
pub struct ColliderArtifactView<'a> {
    pub contract: ContractDigest,
    pub source_frontier: FrontierPosition,
    pub source_root: CanonicalHash,
    pub request_digest: CanonicalHash,
    pub artifact_hash: CanonicalHash,
    pub volume_count: u32,
    pub record_count: u32,
    pub bytes: &'a [u8],
}
pub struct SnapshotMetadata {
    pub uncompressed_bytes: u64,
    pub digest: BlobDigest,
}
pub struct PreparedParticipantState {
    pub opaque: Arc<dyn Any + Send + Sync>,
    pub state_bytes: u64,
    pub commitment: CanonicalHash,
    pub rng_state_digests: BoundedVec<CanonicalHash>,
    pub snapshot: Option<SnapshotMetadata>,
}
pub struct ParticipantStateSink { /* private one-result completion */ }
pub struct ParticipantCompletionSink { /* private state/effect/event completion */ }
pub struct ParticipantSnapshotSink { /* private sequential byte completion */ }

pub enum ParticipantCompletionDisposition {
    Accepted,
    AlreadyCompleted,
    Cancelled,
    LateGeneration,
}

impl ParticipantStateLease {
    pub fn metadata(&self) -> ParticipantTokenMetadata;
    pub fn downcast_ref<T: Any + Send + Sync>(&self) -> Option<&T>;
}
impl ParticipantReplayLease {
    pub fn first_tick(&self) -> Option<Tick>;
    pub fn last_tick(&self) -> Option<Tick>;
    pub fn len(&self) -> u32;
    pub fn total_bytes(&self) -> u64;
    pub fn is_empty(&self) -> bool;
    pub fn record(&self, index: u32) -> Option<ParticipantReplayRecordView<'_>>;
    pub fn records(
        &self,
    ) -> impl ExactSizeIterator<Item = ParticipantReplayRecordView<'_>> + '_;
}
impl ColliderArtifactLease {
    pub fn view(&self) -> ColliderArtifactView<'_>;
}

impl ParticipantStateSink {
    pub fn complete(
        self,
        state: PreparedParticipantState,
    ) -> ParticipantCompletionDisposition;
    pub fn fail(self, error: ParticipantError) -> ParticipantCompletionDisposition;
}
impl ParticipantCompletionSink {
    pub fn push_effect(&mut self, effect: ParticipantEffect)
        -> Result<(), ParticipantOutputError>;
    pub fn push_event(&mut self, event: ParticipantEvent)
        -> Result<(), ParticipantOutputError>;
    pub fn complete(
        self,
        state: PreparedParticipantState,
    ) -> ParticipantCompletionDisposition;
    pub fn fail(self, error: ParticipantError) -> ParticipantCompletionDisposition;
}
impl ParticipantSnapshotSink {
    pub fn write(&mut self, bytes: &[u8]) -> Result<(), ParticipantOutputError>;
    pub fn complete(self, digest: BlobDigest) -> ParticipantCompletionDisposition;
    pub fn fail(self, error: ParticipantError) -> ParticipantCompletionDisposition;
}

pub struct ParticipantEffect {
    pub local_sequence: u32,
    pub command: ParticipantCommand,
}
pub enum ParticipantCommand {
    Erase {
        volume: VolumeId,
        target: MutationShapeQ,
        mode: EraseMode,
        expected_revision: Option<VolumeRevision>,
        expected_source_hash: Option<CanonicalHash>,
    },
    Place {
        volume: VolumeId,
        target: MutationShapeQ,
        cell: CellWire,
        expected_revision: Option<VolumeRevision>,
        expected_source_hash: Option<CanonicalHash>,
    },
    Patch {
        volume: VolumeId,
        cells: BoundedVec<PatchCell>,
        expected_revision: Option<VolumeRevision>,
        expected_source_hash: Option<CanonicalHash>,
    },
    SetPlacement {
        volume: VolumeId,
        placement: PlacementQ,
        expected_revision: Option<VolumeRevision>,
        expected_source_hash: Option<CanonicalHash>,
    },
}
pub enum ParticipantOutputError {
    CountExceeded,
    BytesExceeded,
    DuplicateSequence,
    WrongSchema,
    WrongSource,
    AlreadyTerminal,
    Cancelled,
}
```

The callback form avoids imposing Tokio or a `Send` future. Calls are
nonblocking; completion sinks accept exactly one result and reject wrong
participant/tick/source token/hash, duplicate completion, excessive
bytes/effects, or a closed device generation. `ParticipantStateSink` and
`ParticipantCompletionSink` accept an opaque immutable
`PreparedParticipantState` token bound as specified by TECH-016; the adapter
can inspect a later lease to its own concrete token through `downcast_ref`,
but consumers and Moria never invoke the downcast or interpret the value.
Moria validates `state_bytes`, strategy-specific `snapshot`, commitment, RNG
count, and every descriptor limit before accepting the state. Snapshot
strategy states require matching metadata; reconstructible states require
`snapshot: None`.
There is no `commit` callback: installing the immutable token is the
coordinator's infallible `FrontierBundle` pointer swap. Dropping an uninstalled
token is abort. The descriptor fixes contract/input versions, strategy,
maximum input/effect/snapshot/artifact/state-token bytes, canonical RNG
contracts, and failure policy at genesis.

`prepare_genesis` must return a token whose `ParticipantTokenMetadata.frontier`
is exactly the constructed world's `FrontierPosition::Genesis`.
`prepare_tick` receives both the attempted `tick` and its distinct
`source_frontier`; admission requires
`source_frontier.next_tick() == Some(tick)`, the source lease metadata and
collider view to carry that same position/root, and the produced token to
carry `FrontierPosition::Confirmed(tick)`. Snapshot restore preserves the
request frontier position exactly. Reconstruction starts at
`request.start.position` and produces `Confirmed(request.end_tick)`. These
checks use the closed frontier encoding and never map Genesis to a tick
sentinel.

```rust
pub struct ParticipantDescriptor {
    pub id: ParticipantId,
    pub contract: ContractDigest,
    pub input_schema: SchemaDigest,
    pub event_schemas: BoundedVec<SchemaDigest>,
    pub representations: BoundedVec<ParticipantRepresentationContract>,
    pub strategy: ParticipantRollbackStrategy,
    pub rng: BoundedVec<ParticipantRngContract>,
    pub limits: ParticipantLimits,
    pub failure: ParticipantFailurePolicy,
}

pub struct ParticipantRepresentationContract {
    pub representation_id: u32,
    pub quantity_schema: SchemaDigest,
    pub representation_contract: ContractDigest,
}

pub enum ParticipantRollbackStrategy {
    PerTickSnapshot { max_bytes: u64 },
    ReconstructibleFromCanonicalStateAndLog { max_replay_ticks: u32 },
}

pub struct ParticipantRngContract {
    pub stream: RngStreamId,
    pub algorithm_id: [u8; 16],
    pub algorithm_version: u32,
    pub algorithm_contract: ContractDigest,
    pub state_schema: SchemaDigest,
    pub seed: BoundedBytes64,
}

pub enum ParticipantFailurePolicy {
    NoAdvanceExplicitRetry, // canonical descriptor tag 0
    FailWorld,              // canonical descriptor tag 1
}

pub struct ParticipantLimits {
    pub input_bytes_per_tick: u32,
    pub effects_per_tick: u32,
    pub effect_bytes_per_tick: u32,
    pub events_per_tick: u32,
    pub event_bytes_per_tick: u32,
    pub bytes_per_event: u32,
    pub state_token_bytes: u32,
    pub snapshot_bytes: u32,
    pub artifact_records: u32,
    pub artifact_bytes: u32,
}

pub struct ParticipantEvent {
    pub participant: ParticipantId,
    pub local_sequence: u32,
    pub schema: SchemaDigest,
    pub payload: BoundedBytes,
}
```

RNG entries are sorted by stream ID and follow TECH-016. An adapter declaring
no entries attests that no randomness can affect its canonical state or
effects; Moria provides no implicit stream.

Representation entries are sorted by nonzero `representation_id`, unique, and
bounded by
`ResourceBudgets.identity.representation_contracts_per_participant`. A
participant with no non-placement physical quantities supplies an empty list.
Moria treats each referenced contract as opaque but binds the complete list
into genesis, hashing, persistence, and replay as required by TECH-016.
Participant-owned schemas may not rely on an implicit conversion to or from
the world's placement format.

`event_schemas` is sorted/unique and bounded by `limits.events_per_tick`.
Every emitted event must name one of these genesis-bound opaque schemas; an
empty list means the participant emits no consumer events. The sink assigns
`participant` from the registered adapter; consumer code cannot spoof it.

`ParticipantFailurePolicy` is a closed, genesis-bound choice. It applies only
after registration/admission; malformed descriptors always fail configuration.
There is no skip, stale-token reuse, empty-commitment, CPU-fallback, or
best-effort variant. Unknown wire tags are rejected during descriptor
decoding. Its exact behavior is:

| Failure site | `NoAdvanceExplicitRetry` | `FailWorld` |
| --- | --- | --- |
| Genesis preparation | Fail construction; no genesis is published | Same |
| Ordinary tick preparation/validation | Terminally fail that tick receipt as `FailedNoAdvance { cause: Participant { participant: id, code }, ... }`; retain the attempted tick's source frontier and `Ready`; only a new explicit submission may retry | Same `FailedNoAdvance`, then enter `Failed` with the source frontier as last trustworthy frontier |
| Retained rollback/correction | Fail the correction, drain private tokens, retain the original live bundle and `Ready` | Same atomic abort, then enter `Failed` |
| Durable restore/reconstruction | Fail construction; no restored bundle is published | Same |
| Device generation loss | Fail the affected attempt and enter `RecoveringParticipant`; reject ticks until an explicit bounded recovery recreates an equal-commitment token from a retained snapshot or durable replay bytes | Enter `Failed` immediately; drain old-generation work without publication |
| Snapshot export/checkpoint | Fail only the checkpoint because the installed token remains trustworthy; explicit checkpoint retry is allowed | Same; persistence failure does not corrupt canonical participant state |
| Shutdown | Close new participant permits, suppress publication, drain submitted uses, and report abandonment | Same |

Each explicit retry is one newly admitted, resource-reserved operation; Moria
performs no timer-driven or unbounded internal retry. Recovery installs a
replacement token only when participant ID, contract, frontier position, root, commitment,
RNG commitments, and new device generation exactly match the retained
frontier. It changes no canonical bytes and emits a lifecycle observation.
Recovery mismatch follows the same policy row again. A non-retryable adapter
contract violation or proven commitment divergence is reported distinctly;
the policy still selects operation-scoped versus world-terminal handling but
never permits publication.

CPU participants receive immutable canonical phase-zero input bytes and a
bounded collider artifact already keyed to the attempted tick's
`SourceState(n)`; they do not receive
cell storage. `ParticipantCompletionSink` is one Moria-owned, pre-reserved
completion containing a distinct state token, a fixed-capacity effect sink,
and a fixed-capacity event sink. Participant effects are `Erase`, `Place`,
`Patch`, or `SetPlacement` values with normal preconditions. Events are
participant-owned opaque schema/payload records exposed to the consumer; Moria
validates only their declared schema digest, local sequence, exact length,
zeroed unused capacity, and aggregate limits. They are sorted by
`(ParticipantId, local_sequence)`, committed in the participant outcome bytes
and participant commitment, retained in replay, and returned only in
`TickConfirmed` after the containing tick confirms. They are deliberately not
TECH-025 observations, which remain facts about Moria-owned state under
REQ-012. The substrate assigns no physics, collision-response, damage, or
gameplay meaning to participant events.

`ParticipantStateLease`, `ParticipantReplayLease`, and
`ColliderArtifactLease` are non-`Clone`, immutable owned leases whose backing
storage remains pinned until the lease and its operation sink are both
terminal. Their accessors borrow only for the lease lifetime and allocate
nothing. Moria invokes an adapter only with the registered participant's
token, a contiguous replay range in increasing tick order, and a collider view
whose header and checksum were validated under TECH-053. A wrong concrete
state type returns `None`; the adapter must report participant failure and
cannot request a Moria downcast or fallback. Cancellation closes output sinks
but does not invalidate memory already borrowed by an active callback; late
completion is rejected, and dropping the final lease releases its fixed
permit. CPU leases have no device-generation invalidation. A GPU participant
uses the generation-scoped binding types in TECH-054 instead of these CPU
leases.
An empty replay lease is valid only when the requested reconstruction range
contains no post-start ticks; it reports `len() == 0`, `total_bytes() == 0`,
and `None` for both endpoint accessors. Nonempty endpoints equal the first and
last record views exactly.

The v1 participant model is deliberately one-phase:

- same-tick dependencies between participants are rejected at registration;
  descriptors have no dependency field, participant A cannot read participant
  B's current attempt, and all participants read only `SourceState(n)`, their own
  source token, phase-zero input, and source-bound artifacts;
- bounded participant event output is supported only through the event sink
  above; there is no handoff buffer, prior-feedback ABI, or event-driven second
  participant pass in the same tick; and
- conflicts between effects are not a separate participant mechanism.
  Effects enter TECH-011 phase 4 in `(ParticipantId, local_sequence)` order,
  use ordinary preconditions, and compose exactly like consumer commands. A
  later effect observes the staged result of an earlier effect; an unmet
  precondition produces that effect's ordinary canonical failure rather than a
  DAG, conflict callback, solver, or retry.

This is sufficient for an external CPU or GPU physics/damage implementation to
receive tick-stamped input and canonical collision/occupancy, preserve its
state in the installed token, request admitted effects, and expose its own
simulation/contact events without Moria acquiring its vocabulary.
`prepare_tick` may read only its source lease and must return a distinct token.
Snapshot bytes remain opaque to Moria but their size, digest, retention,
export, durable storage, and staged restoration result are coordinated. For
the reconstructible strategy, `reconstruct` receives bounded canonical
replay-record bytes (from pinned memory for recent correction or
digest-verified checkpoint blobs after restart) and returns a staged token
whose per-tick commitments, event bytes, and effects must match.

The Bevy GPU adapter has equivalent semantics and is specified in
[collision-presentation.md](collision-presentation.md). Registering both CPU
and GPU implementations for one participant ID is rejected; runtime fallback
cannot change which algorithm is authoritative.

### TECH-030 — Simulation-domain canonical union

Implements: REQ-009, REQ-014, REQ-018, REQ-031, REQ-040, REQ-043

An activation input names a live volume, a half-open brick-aligned local region,
base content root, optional per-brick manifest subtree digest, and an activity
class opaque `u32` owned by the consumer. Deactivation names the same canonical
key. Moria stores normalized disjoint intervals per `(volume, activity_class)`.
It computes union by stable endpoint sort (`start` before `end` is versioned)
and emits each covered brick exactly once.

Activation preflight requires all exact content and canonical collision inputs
pinned before batch admission. A missing/mismatched dependency returns
`DependencyNotReady` before admission; corruption detected after admission
causes tick `FailedNoAdvance`. Interest, camera, presentation, I/O completion,
and cache eviction cannot add or remove a region. The normalized union, content
commitments, and activity classes are hashed and retained in rollback.
