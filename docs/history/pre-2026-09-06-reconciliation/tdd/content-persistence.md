# Content, persistence, replay, and reclamation

Base content belongs to the consumer. Moria makes its identity and bounded
materialization explicit, stores only scars and continuation state, and fails
restore unless exact reconstruction is provable.

## Base content

### TECH-041 — Exact base-authority contract

Implements: REQ-004, REQ-008, REQ-014, REQ-020, REQ-021

Every genesis volume selects one:

```rust
pub struct BaseContentSourceId(u32);
pub struct BaseAuthorityId(u32);
pub struct ContentBlobStoreId(u32);
pub struct CheckpointStoreId(u32);
pub struct ReplaySinkId(u32);
pub struct BaseRequestId(u64);
pub struct CheckpointKey([u8; 32]);
pub struct ReplayStreamKey([u8; 32]);
pub struct ContentLineage([u8; 16]);

pub enum BaseAuthority {
    Uniform {
        cell: CellWire,
        identity: ContentDigest,
    },
    Reconstructable {
        lineage: ContentLineage,
        manifest_root: ContentDigest,
        source: BaseContentSourceId,
    },
    Bundled {
        lineage: ContentLineage,
        manifest_root: ContentDigest,
        store: ContentBlobStoreId,
    },
}
```

Each provider ID has `try_from_raw(u32) -> Result<Self, NewtypeValueError>` and
`get(self) -> u32`; it rejects zero and values above `0x7fff_ffff`.
`BaseRequestId` is Moria-created and exposes only `get(self) -> u64`.
`CheckpointKey` has infallible `from_bytes`, `as_bytes`, and `to_bytes`
methods. `ReplayStreamKey::try_from_bytes([u8; 32])` rejects the all-zero
reserved key with `NewtypeValueError::AllZeroReserved` and otherwise preserves
all bytes; it also has `as_bytes` and `to_bytes`. `ContentLineage` has
infallible `from_bytes`, `as_bytes`, and `to_bytes`. Fields are private, every
type is `Copy + Eq + Ord + Hash`, and there is no unchecked tuple constructor.

`lineage` is a semantic family label; `manifest_root` is the exact identity.
Matching lineage alone is never sufficient. The manifest is a canonical
Merkle radix tree from `(base authority id, local brick coordinate)` to a
content digest or uniform cell. A consumer-owned generator may implement
`BaseContentSource`, but its algorithm is opaque and not shipped by Moria.

```rust
pub trait BaseContentSource: Send + Sync + 'static {
    fn descriptor(&self) -> BaseSourceDescriptor;
    fn request(
        &self,
        request: BaseBrickRequest,
        completion: BaseBrickCompletion,
    );
}

pub trait ContentBlobStore: Send + Sync + 'static {
    fn descriptor(&self) -> ContentBlobStoreDescriptor;
    fn get_blob(&self, digest: BlobDigest, limits: BlobLimits, done: LoadSink);
}

pub struct ContentBlobStoreDescriptor {
    pub id: ContentBlobStoreId,
    pub contract: ContractDigest,
    pub max_blob_bytes: u64,
}

pub struct BaseSourceDescriptor {
    pub id: BaseContentSourceId,
    pub contract: ContractDigest,
    pub max_requests_in_flight: u32,
    pub diagnostic_schema: u32,
}

pub struct BaseBrickRequest {
    pub request_id: BaseRequestId,
    pub world: WorldId,
    pub volume: VolumeId,
    pub brick: BrickCoord,
    pub root: CanonicalHash,
    pub expected: BaseBrickExpected,
}

pub enum BaseBrickExpected {
    Uniform { cell: CellWire, digest: ContentDigest },
    CanonicalBrick { bytes: u32, digest: ContentDigest },
}

pub struct BaseSourceFailure {
    pub code: u32,
    pub retryability: Retryability,
    pub diagnostic: BoundedUtf8<96>,
}

pub enum BaseCompletionDisposition {
    Accepted,
    AlreadyCompleted,
    Cancelled,
    LateGeneration,
}

pub enum BaseCompletionWriteError {
    TooLong,
    Cancelled,
    AlreadyTerminal,
}

pub struct BaseBrickCompletion { /* private Moria completion-cell token */ }

impl BaseBrickCompletion {
    pub fn write(&mut self, bytes: &[u8])
        -> Result<(), BaseCompletionWriteError>;
    pub fn finish_brick(self) -> BaseCompletionDisposition;
    pub fn finish_uniform(self, cell: CellWire) -> BaseCompletionDisposition;
    pub fn fail(self, failure: BaseSourceFailure) -> BaseCompletionDisposition;
}
```

A request is an owned immutable descriptor with no borrowed Moria storage. It
names one expected manifest brick/uniform digest and exact encoding length;
`CanonicalBrick.bytes` must equal 2,048 in v1. Moria copies and freezes
`BaseSourceDescriptor` at registration and retains the consumer's `Arc` only
for invocation. The source may retain its request value for diagnostics, but
the values do not pin a root; only Moria's admitted operation record does.

Before invoking consumer code, Moria reserves one callback-completion slot,
one request/lifetime record, and the worst-case 2,048 payload bytes from
`ResourceBudgets`, even for a uniform result. `BaseBrickCompletion` is a
non-`Clone` generation/attempt token into that Moria-owned bounded sink. It
does not own a `Vec` or expose a raw slice. `write` copies sequential bytes
under the sink lock, advances one checked cursor, rejects a write past 2,048,
and returns `Cancelled` after logical cancellation. It accepts no sparse
offsets, overlap, grow operation, writer trait, or consumer allocator.
`finish_brick` succeeds only when the cursor is exactly 2,048;
`finish_uniform` succeeds only for the expected uniform encoding and after no
brick bytes were written. Moria then validates domain, every `CellWire`, exact
length, and digest before residency. Wrong bytes fail the region.

Exactly one callback invocation and one accepted terminal completion exist per
admitted request. A terminal method consumes the token; the backing atomic
cell still rejects a duplicate/forged attempt as `AlreadyCompleted`.
Completion after cancellation or device-generation closure is
`Cancelled`/`LateGeneration`, releases no result into materialization, and
cannot revive the request. Dropping a live completion without a terminal call
records `ProducerDropped`, fails the request, and releases its reservations.
Cancellation before invocation removes the queue entry and never calls the
source. Cancellation after invocation closes the cell; because `write` only
copies while holding the cell lock, Moria can reclaim the sink once the active
copy returns even if consumer code keeps the small closed token.

The only diagnostic crossing the boundary is the fixed code, retryability,
and at most 96 UTF-8 bytes in `BaseSourceFailure`; panic payloads, error
chains, strings, maps, and consumer-owned byte collections are converted to a
bounded `SourcePanicked` diagnostic or dropped. No source call is
automatically retried. A new explicit materialization/interest retry allocates
a new request ID, sink, and permit; timing never substitutes content.

If a source cannot promise reconstructability, the consumer must use
`Bundled`, whose store contains every base blob referenced by the manifest.

### TECH-042 — Materialization and scar overlay

Implements: REQ-001, REQ-004, REQ-009, REQ-014, REQ-016, REQ-018

Materialization pins an immutable root, resolves each requested brick's base
digest, obtains and verifies its canonical base payload, then overlays at most
one scar leaf from that root. The resulting complete brick enters the
noncanonical resident cache keyed by `(world root, volume, brick, source
digest)`. It becomes `Ready` only after GPU upload and directory publication
complete.

Scar leaves always contain a complete replacement brick or uniform cell; they
never depend on edit order during restore. Mutation compares each new complete
brick with the verified base payload. Equal bricks remove the scar key;
different bricks replace it. This keeps untouched/homogeneous space cheap and
makes scar restore order-independent.

A committed dirty scar remains reachable through canonical roots regardless of
resident-cache interest. Eviction may remove a rematerialized cache brick but
not the scar leaf, rollback node, dirty journal reference, or base identity.

## Durable checkpoint format

### TECH-043 — Checkpoint store and atomic commit

Implements: REQ-005, REQ-014, REQ-015, REQ-021

Persistence is store-neutral:

```rust
pub trait CheckpointStore: Send + Sync + 'static {
    fn descriptor(&self) -> CheckpointStoreDescriptor;
    fn put_blob(&self, digest: BlobDigest, bytes: OwnedBytes, done: StoreSink);
    fn get_blob(&self, digest: BlobDigest, limits: BlobLimits, done: LoadSink);
    fn load_manifest(
        &self,
        key: CheckpointKey,
        limits: BlobLimits,
        done: ManifestLoadSink,
    );
    fn commit_manifest(
        &self,
        key: CheckpointKey,
        manifest: OwnedBytes,
        done: CommitSink,
    );
}

pub struct CheckpointStoreDescriptor {
    pub id: CheckpointStoreId,
    pub contract: ContractDigest,
    pub max_blob_bytes: u64,
    pub max_manifest_bytes: u64,
    pub atomic_manifest_visibility: bool,
}

pub struct BlobLimits {
    pub max_bytes: u64,
    pub expected_bytes: Option<u64>,
}

pub struct StoreFailure {
    pub code: StoreErrorCode,
    pub retryability: Retryability,
    pub diagnostic: BoundedUtf8<96>,
}

pub enum StoreErrorCode {
    Unavailable,
    PermissionDenied,
    Capacity,
    Corrupt,
    NotFound,
    UnsupportedAtomicCommit,
    Internal,
}

pub enum StoreCompletionDisposition {
    Accepted,
    InvalidCompletion,
    AlreadyCompleted,
    Cancelled,
    LateGeneration,
}

pub enum StoreCompletionWriteError {
    TooLong,
    AlreadyTerminal,
    Cancelled,
}

pub struct StoreSink { /* private put-blob completion token */ }
pub struct LoadSink { /* private get-blob completion token */ }
pub struct ManifestLoadSink { /* private key-load completion token */ }
pub struct CommitSink { /* private manifest-commit completion token */ }

impl StoreSink {
    pub fn stored(
        self,
        digest: BlobDigest,
        durable_bytes: u64,
    ) -> StoreCompletionDisposition;
    pub fn fail(self, failure: StoreFailure) -> StoreCompletionDisposition;
}

impl LoadSink {
    pub fn write(&mut self, bytes: &[u8])
        -> Result<(), StoreCompletionWriteError>;
    pub fn finish(
        self,
        digest: BlobDigest,
    ) -> StoreCompletionDisposition;
    pub fn fail(self, failure: StoreFailure) -> StoreCompletionDisposition;
}

impl ManifestLoadSink {
    pub fn write(&mut self, bytes: &[u8])
        -> Result<(), StoreCompletionWriteError>;
    pub fn finish(
        self,
        key: CheckpointKey,
    ) -> StoreCompletionDisposition;
    pub fn fail(self, failure: StoreFailure) -> StoreCompletionDisposition;
}

impl CommitSink {
    pub fn committed(
        self,
        key: CheckpointKey,
        manifest_digest: BlobDigest,
    ) -> StoreCompletionDisposition;
    pub fn fail(self, failure: StoreFailure) -> StoreCompletionDisposition;
}
```

Calls are asynchronous callbacks with explicit byte limits. Blob keys are
BLAKE3 digests of uncompressed canonical bytes; zstd is a storage encoding
whose version/options are recorded and whose decode has a maximum output.
Deduplication is optional and cannot change semantics.

Every `StoreSink`, `LoadSink`, `ManifestLoadSink`, and `CommitSink` is a
non-`Clone` token into the
same bounded completion-cell discipline as TECH-041. Moria reserves the
callback slot, worst-case result/diagnostic bytes, and operation record before
calling the store. Load sinks accept sequential bounded copies into a
Moria-owned `BlobLimits.max_bytes` buffer, not an `OwnedBytes` returned by the
store. `expected_bytes` must be `Some(n)` with `n <= max_bytes` whenever the
caller has a manifest descriptor or content record that declares an exact
length. It is `None` only for the initial key-based manifest lookup, whose
encoded length cannot be known before reading. Freeze/admission rejects
`Some(n)` above `max_bytes`.

Every `get_blob` call uses `Some(expected_bytes)`, and `LoadSink::finish`
requires the requested digest and a cursor exactly equal to that value.
For `ManifestLoadSink`, whose request alone uses `expected_bytes: None`,
`finish` requires the requested key and an actual cursor no greater than
`max_bytes`; the complete manifest header, declared encoded length, trailing
checksum, and absence of trailing bytes are then validated before any manifest
field is used. A truncated or empty manifest therefore cannot succeed merely
because it is shorter than the maximum. Every manifest-referenced scar,
snapshot, or replay blob load uses its descriptor's exact uncompressed
canonical length; content bricks use the exact length from
`BaseBrickExpected`. Store-side compression is decoded before bytes enter the
sink and cannot change this length contract.

Known-length short output, framing/digest mismatch, identity mismatch, or a
wrong durable-byte count returns `InvalidCompletion` and fails the owning
operation. A write beyond `max_bytes` or beyond a present `expected_bytes`,
write after terminal, or write after cancellation returns the closed
`StoreCompletionWriteError` variants `TooLong`, `AlreadyTerminal`, or
`Cancelled`. `stored` and `committed` echo the expected identity.
Commit/put completions carry only the closed status and bounded diagnostic.
Duplicate, dropped, cancelled, and late completions have the explicit
dispositions above and cannot commit a manifest.
`OwnedBytes` passed *to* `put_blob`/`commit_manifest` is immutable Moria-owned
input whose lifetime is pinned through completion; the store receives no
allocator or growable Moria collection.
Builder freeze rejects a checkpoint-store descriptor whose
`atomic_manifest_visibility` is false or whose declared maxima cannot satisfy
the configured checkpoint limits.

`CheckpointKey` is a consumer-selected fixed 32-byte opaque key scoped to one
registered `CheckpointStoreId`. It is the store-visible locator for the
atomically visible manifest; it is not a blob digest. `load_manifest` is the
only key-to-manifest operation and returns `NotFound` for an invisible,
uncommitted, or absent key. A successful `CommitSink::committed` means a later
`load_manifest` on that store and key observes the complete exact manifest
bytes or a store failure—never an older partial write. Recommitting an existing
key with different bytes is rejected as `Corrupt`; idempotent recommit of the
same manifest digest may succeed.

Dropping any live sink reports `ProducerDropped`; cancellation closes it and
waits only for an active bounded copy to leave the cell; old-generation
success is `LateGeneration` and releases lifetime state without delivering
bytes. Store calls are never automatically retried. A new checkpoint, restore,
recovery, or bundled-content operation owns the explicit retry and a fresh
sink/reservation.

The native reference store writes content-addressed blobs first, verifies
length/digest, fsyncs them, writes a manifest to a unique temporary name,
fsyncs it, atomically renames to the checkpoint key, and fsyncs the parent
directory. Only a committed manifest makes a checkpoint visible. A crash or
failure before rename leaves ignorable orphan blobs, never a partial
checkpoint. Store adapters must document equivalent atomicity or report
`UnsupportedAtomicCommit`.

### TECH-044 — Versioned checkpoint manifest

Implements: REQ-001, REQ-014, REQ-017, REQ-028, REQ-032

`moria-checkpoint-v1` is a canonical manifest containing:

- product, canonical encoding, arithmetic, hash, transition, and persistence
  schema digests;
- exact per-world configuration fingerprint and its
  `PlacementFixedFormat { fractional_bits, cell_extent_raw, simulation_unit }`;
- checkpoint-store ID, store contract digest, and store-visible checkpoint key;
- world/genesis ID, exact base lineage and manifest roots, material registry;
- confirmed and durable tick, world root hash, per-volume revisions;
- live and retired volume identities, domains, kinds, placements, and scar
  root hashes;
- content-addressed scar radix node and brick-blob descriptors
  `{kind, uncompressed_bytes, blob_digest}`;
- `next_volume_serial` and simulation-domain normalized intervals;
- participant IDs/contracts/input schemas/strategies/commitments, complete
  participant representation contracts, complete RNG descriptors and current
  RNG-state commitments, and snapshot blob digests with exact uncompressed
  byte lengths where applicable;
- for every reconstructible participant, its required inclusive replay tick
  range and a sorted list of content-addressed
  `moria-checkpoint-replay-v1` chunk descriptors `{first_tick, last_tick,
  record_count, uncompressed_bytes, blob_digest}` whose exact bytes cover the
  union of all such ranges without gaps or overlap;
- the active-history digest over the ordered semantic tick-record projection,
  plus the public stream key, physical durable-prefix position/digest, and
  per-record physical sequence/subrecord locator needed to bind that covered
  range through any `CorrectionBranch` to the public `moria-replay-v1`
  sequence;
- completeness counts, total uncompressed bytes, and manifest checksum.

Derived meshes, dressing, resident base-cache entries, physical slots, Bevy
entities, telemetry, timings, and receipt IDs are forbidden. Unknown required
fields/version tags fail restore. The decoder enforces configured node, blob,
depth, and byte bounds before allocation.

### TECH-045 — Checkpoint snapshot isolation

Implements: REQ-005, REQ-014, REQ-017, REQ-018

```rust
pub struct CheckpointRequest {
    pub world: WorldId,
    pub store: CheckpointStoreId,
    pub key: CheckpointKey,
    pub frontier: FrontierSummary,
    pub max_uncompressed_bytes: u64,
    pub max_manifest_nodes: u32,
    pub max_manifest_blobs: u32,
}

pub struct DurableFrontier {
    pub store: CheckpointStoreId,
    pub key: CheckpointKey,
    pub frontier: FrontierSummary,
}

pub struct ParticipantCommitmentFact {
    pub participant: ParticipantId,
    pub contract: ContractDigest,
    pub state_bytes: u64,
    pub state_digest: CanonicalHash,
    pub rng_state_digests: BoundedVec<CanonicalHash>,
}

pub struct CheckpointCommitted {
    pub durable: DurableFrontier,
    pub volume_revisions: BoundedVec<VolumeRevisionFact>,
    pub participant_commitments: BoundedVec<ParticipantCommitmentFact>,
    pub scar_nodes: u32,
    pub scar_blobs: u32,
    pub participant_snapshot_blobs: u32,
    pub replay_records: u32,
    pub replay_chunks: u32,
    pub compressed_bytes: u64,
    pub uncompressed_bytes: u64,
    pub manifest_digest: BlobDigest,
}
```

A checkpoint request names an exact
`FrontierPosition::Confirmed(tick)` and root hash, a maximum readback/store
budget, and one exact frozen-registry `CheckpointStoreId`. A genesis-position
checkpoint is rejected because no numbered tick is confirmed yet.
Unknown stores reject admission and no fallback to the configured default or
another registered store occurs. Admission pins that immutable root and participant
frontier, reserves one queue/operation/receipt record and the declared staging
and terminal-result bytes, and returns the TECH-070 `CheckpointReceipt`.
Admission rejection returns the unchanged `CheckpointRequest`. New ticks may
confirm concurrently; they remain dirty relative to the checkpoint.

Canonical substrate traversal follows scar and metadata nodes only; participant
snapshots use the separate export path below. GPU blobs are copied
through bounded staging slots in lexicographic logical-key order, mapped,
decoded, digest-verified, and handed to the store. It never scans untouched
base cells or serializes derived state. Completion reports exactly:

```text
checkpoint key, durable tick/root, per-volume revisions,
participant commitments, scar/node/blob and participant-snapshot-blob counts,
replay-record/chunk counts, compressed/uncompressed bytes
```

The pinned participant frontier is part of traversal. For every
`PerTickSnapshot` participant in `ParticipantId` order, checkpoint admission
reserves its declared `max_bytes` against both the participant-export pool and
the checkpoint's total byte budget. Moria invokes the public CPU/GPU snapshot
export operation on the exact pinned state token. Export state is:

```text
Pinned -> ExportReserved -> Exporting -> BytesVerified
       -> BlobPutPending -> BlobDurable -> ManifestReferenced
       \-> Failed
```

The export result is bound to participant ID/contract, pinned tick/root,
participant commitment, snapshot length/digest, and device generation where
applicable. Moria checks all fields, exact length, and BLAKE3 digest before
calling `CheckpointStore::put_blob`; the store completion must confirm the
same digest. At most the configured three aggregate checkpoint staging slots
and `CheckpointBudgets.store_bytes_in_flight` (default 64 MiB, portable maximum
256 MiB), including scar, participant, and replay bytes, are in flight.
Snapshot participants cannot provide an external locator instead of bytes.
Reconstructible participants contribute descriptor, commitment, and required
replay range but no snapshot blob. Moria takes the union of those ranges,
pins every exact confirmed tick record from the active semantic projection,
and rejects
checkpoint admission if any record is absent or a participant's range exceeds
its declared `max_replay_ticks`. Chunks contain at most 64 consecutive records
and at most 8 MiB uncompressed; an oversized individual record is one
single-record chunk only if it fits both the checkpoint request and
`rollback.log_bytes`, otherwise admission fails. Chunk encoding retains the
exact length-prefixed tick-record bytes—extracting the identical framed
subrecord when its physical locator is inside a correction branch—and adds only the
`moria-checkpoint-replay-v1` version, first/last tick, record count, and
checksum. It never reconstructs records from digests.

Replay chunks use the same bounded durable state machine:

```text
LogPinned -> BytesReserved -> ChunkEncoded -> BytesVerified
          -> BlobPutPending -> BlobDurable -> ManifestReferenced
          \-> Failed
```

Encoding and storage reserve the chunk's declared uncompressed bytes before
work starts. Moria verifies tick continuity, each embedded record checksum and
digest, the chunk length, its active-history/physical-sequence locator
binding, and its BLAKE3 blob digest before
`CheckpointStore::put_blob`; store completion must confirm that digest.

`commit_manifest` is not called until every scar/node/metadata,
participant-snapshot, and required replay chunk has reached `BlobDurable`. The
manifest references exactly those verified digests. Any participant export,
generation, mapping, replay gap/encode, store, size, or digest failure enters
`Failed`, publishes no manifest, and cannot report that participant frontier
durable. Submitted exports and puts drain for lifetime safety; the root,
participant tokens, and log records remain pinned until all reservations are
released.

Failure keeps the root and all later dirty truth live, reports whether orphan
blobs may exist, and publishes no manifest. Cancelling before a submitted
readback stops new batches; submitted batches drain before permits/root pins
release.

## Restore

### TECH-046 — Durable restore protocol

Implements: REQ-008, REQ-014, REQ-021, REQ-029, REQ-035

```rust
pub struct RestoreRequest {
    pub store: CheckpointStoreId,
    pub key: CheckpointKey,
    pub limits: RestoreLimits,
}

pub struct RestoreLimits {
    pub manifest_bytes: u64,
    pub blob_count: u32,
    pub uncompressed_bytes: u64,
    pub replay_ticks: u32,
}

pub struct RestoreReady {
    pub frontier: FrontierSummary,
    pub durable_source: DurableFrontier,
    pub next_tick: Tick,
    pub rebuilt_bricks: u32,
    pub replay: ReplayStreamPosition,
}
```

Restore is world construction, not a post-genesis mutation:

1. resolve `request.store` in the frozen builder registry, call that exact
   store's key-based `load_manifest(request.key, BlobLimits {
   max_bytes: limits.manifest_bytes, expected_bytes: None }, ...)`, and
   bounded-decode the complete framed returned manifest;
2. verify all contract versions and manifest digest;
3. match material, base lineage **and exact manifest roots**, volume sources,
   configuration fingerprint, placement format, and participant registrations;
4. load every referenced scar/node/snapshot/replay-chunk blob with
   `expected_bytes: Some(manifest_descriptor.uncompressed_bytes)`, enforce
   bounds, decompress in the store adapter before sink delivery, and verify its
   uncompressed digest; validate each replay
   chunk's tick interval, embedded record checksum/digest, exact continuity,
   and prefix/suffix digest against the manifest before exposing a
   `ParticipantReplayLease`;
5. rebuild new-generation GPU pages and radix roots from logical keys;
6. ask snapshot participants to create staged tokens from verified snapshot
   bytes and reconstructible participants to create staged tokens from only
   the restored canonical frontier plus their manifest-declared replay range,
   while proving every reproduced intermediate commitment and the saved final
   participant/RNG-state commitments;
7. recompute every root bottom-up and compare the saved world root;
8. encode one `Checkpoint`-anchored `moria-replay-v1` header containing the
   exact request store/key, verified manifest digest,
   `FrontierPosition::Confirmed(saved_tick)`, and checked
   `next_tick = saved_tick + 1`; append it as sequence zero to the builder's
   selected replay stream and wait for matching durable completion; and
9. publish the restored frontier and readiness context, then permit the next
   tick at replay sequence one.

The restored GPU root and all participant tokens remain in one private genesis
bundle until all steps pass; adapter calls never mutate a live singleton.
Publishing that bundle is one pointer swap. Corruption, missing IDs, replay
gaps, missing blobs, unsupported contract, wrong lineage/root, content-source
inability, participant mismatch, resource exhaustion, unsupported device
capability, or canonical-math/hash disagreement fails the entire restore.
Migration/rebase is a separate
consumer-authored tool that produces a new genesis identity; Moria does not
guess or mutate an old checkpoint in place.

`WorldBuilder::restore_checkpoint` consumes the private builder and request
only on admission. Rejection returns both in `RestoreRejected`; acceptance
returns `RestoreReceipt`. Cancellation/failure destroys the private bundle and
publishes no world. Retry is a new call with a returned or newly constructed
builder; restore never attaches to or replaces an already-published world.
The manifest records its `CheckpointStoreId`; restore rejects a mismatch even
if another registered store happens to return identical bytes.
TECH-017 reserves the builder's replay pair and eventual tombstone before
restore admission. Failure/cancellation before the sequence-zero invocation
releases both; after invocation it drains the sink call and retires the pair.
A sink failure completes `RestoreReceipt` with
`OperationError { code: StoreFailure, scope:
Provider(ReplaySink(configured_id)), committed: None, ... }`, publishes no
world, and never falls back to another sink. `RestoreReady.replay` is
`durable_through_sequence == 0`, `next_sequence == 1`; therefore the first
later confirmed tick can only append after the durable checkpoint anchor.
`RestoreReady.frontier.position` is the saved `Confirmed(tick)`, and
`RestoreReady.next_tick` is the same checked value encoded in that header.

## Replay and rollback

### TECH-047 — Replay record and divergence artifact

Implements: REQ-032, REQ-034, REQ-035, REQ-038, REQ-043

The mandatory live-record export seam is:

```rust
pub trait ReplaySink: Send + Sync + 'static {
    fn descriptor(&self) -> ReplaySinkDescriptor;
    fn append(
        &self,
        request: ReplaySinkRequest,
        bytes: OwnedBytes,
        done: ReplayAppendSink,
    );
}

pub struct ReplaySinkDescriptor {
    pub id: ReplaySinkId,
    pub contract: ContractDigest,
    pub max_record_bytes: u64,
    pub max_records_in_flight: u32,
}

pub struct ReplaySinkRequest {
    pub stream: ReplayStreamKey,
    pub sequence: u64,
    pub range: ReplayAppendRange,
    pub bytes: u64,
    pub digest: BlobDigest,
}

pub enum ReplayAppendRange {
    Header {
        starting: FrontierPosition,
        next_tick: Tick,
    },
    TickRecords {
        first_tick: Tick,
        last_tick: Tick,
        record_count: u32,
    },
    CorrectionBranch {
        target_tick: Tick,
        superseded_through: Tick,
        corrected_through: Tick,
        record_count: u32,
    },
}

pub struct ReplayExportFailure {
    pub sink: ReplaySinkId,
    pub request: ReplaySinkRequest,
    pub failure: ErrorCode,
}

pub struct ReplayAppendSink { /* private completion token */ }

impl ReplayAppendSink {
    pub fn stored(
        self,
        stream: ReplayStreamKey,
        sequence: u64,
        digest: BlobDigest,
    ) -> StoreCompletionDisposition;
    pub fn fail(self, failure: StoreFailure) -> StoreCompletionDisposition;
}

pub struct ReplayRequest {
    pub header: OwnedBytes,
    pub records: BoundedVec<OwnedBytes>,
    pub limits: ReplayLimits,
}

pub struct ReplayLimits {
    pub max_ticks: u32,
    pub max_input_bytes: u64,
    pub max_private_bytes: u64,
    pub max_artifact_bytes: u64,
    pub anchor_restore: Option<RestoreLimits>,
}

pub struct ReplayCompleted {
    pub frontier: FrontierSummary,
    pub ticks: BoundedVec<CorrectedTickOutput>,
    pub replay: ReplayStreamPosition,
}

pub struct ReplayStreamPosition {
    pub stream: ReplayStreamKey,
    pub durable_through_sequence: u64,
    pub durable_prefix_digest: BlobDigest,
    pub next_sequence: u64,
}

pub struct DivergenceArtifact {
    pub format: ContractDigest,
    pub genesis_digest: BlobDigest,
    pub contract_digests: BoundedVec<ContractDigest>,
    pub execution: ExecutionSummary,
    pub earliest_tick: Tick,
    pub input_prefix: OwnedBytes,
    pub expected_root: CanonicalHash,
    pub actual_root: CanonicalHash,
    pub expected_outcome: CanonicalHash,
    pub actual_outcome: CanonicalHash,
    pub expected_participants: BoundedVec<ParticipantCommitmentFact>,
    pub actual_participants: BoundedVec<ParticipantCommitmentFact>,
    pub changed_keys: BoundedVec<CanonicalLogicalKey>,
    pub byte_differences: BoundedVec<CanonicalByteDifference>,
}

pub struct CanonicalLogicalKey {
    pub kind: u8,
    pub key: [u8; 16],
}

pub struct CanonicalByteDifference {
    pub key: CanonicalLogicalKey,
    pub offset: u32,
    pub expected: BoundedBytes64,
    pub actual: BoundedBytes64,
}

pub struct ReplayFailure {
    pub error: OperationError,
    pub divergence: Option<DivergenceArtifact>,
}
```

The registered `ReplaySink` contract is per-record atomic and append-only for
each `(sink, stream, sequence)`: a matching `stored` completion means the
complete exact bytes are durable and visible at that sequence, while `fail`,
drop, or a rejected completion means no record is visible there. Reuse of a
sequence with different bytes is `Corrupt`; replaying the identical digest may
only return the same durable success. This is required equally for ordinary
tick and correction-branch records and is validated when the provider
descriptor is frozen.

`ReplayAppendRange` has wire tag `0 = Header`, `1 = TickRecords`, and
`2 = CorrectionBranch`.
`Header` is valid only at sequence zero, has implicit record count zero, and
requires `next_tick` to equal the checked next-tick function of `starting`.
`TickRecords` is valid only at sequence greater than zero; v1 requires
`record_count == 1` and `first_tick == last_tick`. Unknown tags, a header at a
later sequence, a tick record at sequence zero, an inconsistent next tick, or
an inconsistent tick range fails the owning construction/append before sink
success can publish anything. `CorrectionBranch` is valid only at the current
nonzero `next_sequence`; it requires
`target_tick < superseded_through == corrected_through`, checked
`corrected_through - target_tick <= u32::MAX`,
`record_count == corrected_through - target_tick`, and an exact target
frontier in the current active-history projection. It is one append record,
not `record_count` independent sink calls.

`ReplayStreamKey` is a consumer-selected fixed 32-byte key. Its only live
source is the per-world value passed to `MoriaClient::begin_world` and frozen
in that `WorldBuilder`; every header/tick request and completion uses that
exact value together with the configured `ReplaySinkId`. TECH-017 reserves
duplicate pairs. The diagnostic records above are closed and not storage
handles; unknown logical-key tags are rejected.

`moria-replay-v1` is an appendable sequence. Its sequence-zero header has one
closed replay identity and one closed anchor tag:

```text
ReplayIdentityV1 {
  authority_status: u8,
  configuration_fingerprint: [u8; 32],
}
header {
  common: genesis identity/digest, contract digests, participant descriptors,
          replay_identity: ReplayIdentityV1, placement format,
          starting frontier { world, position, root_hash }, next tick
  anchor:
    Genesis { canonical genesis bytes/digest }
    Checkpoint { checkpoint_store_id, checkpoint_key, manifest_digest,
                 durable_frontier }
}
tick record {
  tick, sealed TickBatch bytes/digest, canonical outcome bytes/digest,
  participant commitments, bounded opaque participant event bytes/digest,
  expected world root hash
}
correction branch record {
  target frontier/root, superseded-through frontier/root,
  previous active-history digest, corrected-through frontier/root,
  corrected active-history digest, replacement record count,
  exact length-prefixed tick records for target+1..=corrected-through
}
```

`ReplayIdentityV1` is exactly 33 bytes with no length prefix or padding.
`authority_status` is `0 = ReplayGrade` or `1 = DiagnosticCandidate`; every
other value is invalid. `configuration_fingerprint` is the exact TECH-009
digest from the frozen builder. Its identity digest, where a digest is needed,
is `BLAKE3("moria/replay-identity-v1" || authority_status:u8 ||
configuration_fingerprint:[u8;32])`.

Genesis writes the status of the frontier it will publish. Checkpoint restore
writes the verified checkpoint status, which v1 requires to be
`ReplayGrade`; public replay requires the source identity's status to equal
the private builder's selected status. Before any canonical transition,
participant callback, device submission, or destination-sink invocation,
restore and public replay compare the two identity fields exactly and compare
the already listed contract/participant/placement fields by their canonical
v1 encodings. Any mismatch is `ContractMismatch`.

Adapter, device, backend, driver, process, worker, and diagnostic fault-plan
context do not occur in the header, any record digest, the active-history
digest, or `durable_prefix_digest`, and are never replay compatibility inputs.
They remain noncomparison fields of `ExecutionSummary` in telemetry and
divergence evidence. Changing only `ExecutionSummary.adapter_context` cannot
change sequence-zero bytes or reject restore/replay. Changing placement,
arithmetic/table data, or any other canonical configuration input changes the
configuration fingerprint and must reject before transition as
`ContractMismatch`.

The physical stream is append-only, while its active history is the unique
semantic projection obtained by folding records in sequence order. A
`TickRecords` item appends to that projection. A `CorrectionBranch` verifies
its target frontier/root and previous active-history digest, removes the
projected suffix after `target_tick`, then appends its embedded replacement
tick records. The superseded physical bytes remain durable evidence but are
not canonical-log inputs after the branch. The active-history digest is
`BLAKE3("moria-active-history-v1" || header_anchor_digest || ordered
(tick:u64 LE, tick_record_digest:[u8;32]))`; branch validation recomputes it
before accepting the record. Unknown, gapped, mismatched, nested, or
overlapping embedded records invalidate the branch. A consumer replaying the
complete physical stream therefore derives exactly one corrected ordered log
without deleting or relabeling old durable bytes.

Records are length-prefixed, checksummed, and bounded. An ordinary tick record
is appended only after that tick confirms; a correction branch is durably
appended before its corrected suffix publishes under TECH-048. Presentation
and timing are excluded. A `Genesis` anchor's
starting frontier is exactly `FrontierPosition::Genesis` and its `next_tick`
is exactly zero. A `Checkpoint` anchor's starting frontier is
`FrontierPosition::Confirmed(t)` and its `next_tick` is exactly checked
`t + 1`; store/key and manifest digest are the exact values verified by
TECH-046. No `Tick` sentinel encodes genesis. A public replay of a checkpoint
stream must supply `ReplayLimits.anchor_restore: Some(...)`, resolve the named
store in its frozen builder, and privately run the same bounded restore before
applying the first tick record. Supplying `Some` for a genesis anchor or
`None` for a checkpoint anchor is `InvalidRequest`.

During private genesis, Moria first appends the header as sequence zero with
`ReplayAppendRange::Header { starting: FrontierPosition::Genesis, next_tick:
Tick::from_raw(0) }` and publishes the pre-tick genesis frontier only after
that completion is durable;
failure fails `GenesisReceipt` and publishes no world. Durable restore performs
the checkpoint-anchor sequence-zero operation specified by TECH-046. Every
sequence-zero request uses `ReplayAppendRange::Header`; it never fabricates a
tick range or `record_count`. Each later tick record uses
`ReplayAppendRange::TickRecords { first_tick: record.tick, last_tick:
record.tick, record_count: 1 }`. Tick zero therefore appears first in this
range only after its batch confirms. A correction uses the one
`CorrectionBranch` append specified by TECH-048; its embedded tick frames use
the identical standalone tick-record encoding. Each append is invoked in
physical sequence order through that registered sink. At most one append for a given
world stream is invoked at once; the
configured in-flight record/byte budgets cover all worlds and providers
without weakening this per-stream order. Moria reserves the immutable
record bytes, one callback cell, and one in-flight record/byte permit before
invocation. `ReplayAppendSink` has the same one-terminal, drop, cancellation,
duplicate, digest, and generation rules as TECH-043. A sink result cannot
change the already-confirmed canonical state. Each tick record remains pinned
until the matching `(stream, sequence, digest)` success;
at the in-memory log boundary later `reserve_tick` returns
`PersistenceBackpressure` rather than overwrite it.

The v1 post-genesis append-failure policy is deliberately terminal rather than
an implicit or public redrive. If `ReplayAppendSink::fail`, producer drop,
wrong identity/digest, or an invalid first terminal completion occurs after a
tick is confirmed, or while a correction branch is awaiting its required
prepublication durability, Moria closes further tick admission and moves the
world from `Ready` to `Failed`. A duplicate call observed after an already
accepted success is only `AlreadyCompleted` and does not fail the world. The
exact undurable record and every earlier required record remain pinned until
shutdown releases the world; a checkpoint does not relabel the failed replay
append as durable. There is no facade method that retries, skips, changes the
stream, or attaches another sink to that published world.

Committed-effect reporting distinguishes the two append sites. For an
ordinary tick record, the tick was already published before its export began:
its already-returned `TickConfirmed` remains valid, and the one
`WorldLifecycleFact.failure` is
`OperationError { code: StoreFailure,
scope: Provider(ReplaySink(id)), retryability: Never,
committed: Frontier(the_confirmed_tick_frontier), ... }`. For a
`CorrectionBranch`, no correction frontier has published: the original
frontier and active-history projection remain installed, and TECH-048 requires
both `CorrectionError.error` and the matching
`WorldLifecycleFact.failure` to carry
`OperationError { code: StoreFailure,
scope: Provider(ReplaySink(id)), retryability: Never,
committed: None, ... }`. In both cases `WorldLifecycleFact.frontier` is the
same last readable trustworthy frontier, and the fact also carries the exact
`ReplayExportFailure`. A
`FailureCounter { code: ErrorCode::StoreFailure, count: ... }` telemetry
bucket and the replay-sink pinned-record/byte/oldest-age counters record the
failure.

Replay append has no consumer cancellation point. Shutdown stops invoking
later appends, waits for or closes the one already-invoked completion cell
under the bounded provider-drain rule, reports the confirmed frontier as
committed but the replay record as undurable, and then releases the pin.
The failure transition appends exactly one `WorldLifecycleFact` whose
`replay_export_failure` is the closed `ReplayExportFailure` above; every other
lifecycle fact carries `None`. `ShutdownReport.replay_export_failure` repeats
that same fixed metadata. Because one stream has at most one invoked append,
both fields are `Option`, not a growable list. The record contains the exact
sink, stream, sequence, append range (header position/next tick, confirmed
tick range/count, or correction target/superseded/corrected/count), byte
length, digest, and failure code from the original request; it does not copy the pinned replay
bytes. Its retained v1 allocation is at most 128 bytes and is reserved in the
observation/terminal-receipt budgets before invoking the append. Shutdown
constructs and retains the report metadata before releasing the raw replay
record bytes and their pin; dropping the report later releases only its
ordinary terminal-receipt record.
Device loss does not cancel a host store call or change its stream/sequence;
its completion remains valid because replay bytes are device-independent.
A completion carrying a closed world attempt/generation may only acknowledge
lifetime release and cannot return the world to `Ready`. Genesis, restore,
and public-replay bootstrap failures retain the stronger construction rule:
no world or ready construction receipt is published.

`ReplayStreamPosition` is the public proof of append ordering and is the only
sequence-prefix digest exposed by `ReplayCompleted`.
`durable_prefix_digest` is BLAKE3 over the ordered tuple stream
`(sequence:u64 LE, record_digest:[u8;32])`, beginning at sequence zero;
`next_sequence` is checked `durable_through_sequence + 1`. Genesis and restore
return `{ durable_through_sequence: 0, next_sequence: 1 }`. Every subsequent
append advances all three fields only after matching sink success; integer
overflow fails the world before another invocation. A correction branch
advances the physical sequence and prefix digest exactly once regardless of
its embedded tick count; its separate active-history digest commits the
semantic suffix replacement.
`ReplayCompleted.frontier` is the last replayed
`FrontierPosition::Confirmed(tick)`, or the header's unchanged starting
position when the owned record sequence is empty.

`ReplaySink` is deliberately write-only from Moria's perspective. The
consumer retrieves its own stored bytes through its own storage API and passes
them back as bounded `OwnedBytes` in `ReplayRequest`; that round trip grants no
access to Moria roots or buffers.

Public replay is the dedicated `WorldBuilder::replay_records` operation, not
ordinary live `submit_tick`. Admission consumes a private builder plus the
owned header and record vector, verifies all count/byte limits against
`ResourceBudgets.rollback`, reserves a private root/participant bundle and the
worst-case result/artifact bytes, reserves the builder's fresh replay
pair/tombstone under TECH-017, and returns `ReplayReceipt`. It also verifies
that every source item fits the selected sink descriptor and aggregate
bootstrap in-flight/byte limits. Rejection
returns the unchanged builder and complete owned request in `ReplayRejected`.
The header must describe the builder's world and frozen registries, and its
`next_tick` must equal the checked next-tick function for its starting frontier
position. `ReplayRequest.records` are the exact physical append records after
sequence zero, in sequence order. Each ordinary tick record must be the next
tick in the active semantic projection. Each branch must satisfy the closed
fold above, must target a still-retained private frontier no deeper than
`rollback.ticks_per_correction`, and must contain exactly the contiguous
replacement ticks through the superseded present. `ReplayLimits.max_ticks`
counts every decoded tick transition, including tick records later
superseded by a branch, while `max_input_bytes` counts all physical and
embedded bytes. These bounds reserve the private rollback deque needed to
install a branch target without whole-world traversal. Every tick subrecord is
individually within the canonical limits and carries the expected
root/outcome/participant digests. Checked tick, sequence, count, or byte
overflow rejects admission. A checkpoint anchor is first restored privately
as described above; a genesis anchor constructs the declared genesis
privately.

Replay decodes each sealed batch into the same canonical transition function,
but only inside the private builder context. For every tick the exclusive
private publication step calculates the candidate root, outcome digest,
participant commitments/events, and expected-hash comparison before advancing
the private replay frontier. On a branch it restores the named retained
private target, discards the superseded private suffix, and processes the
embedded replacement records; only the branch's corrected projection remains
in `ReplayCompleted.ticks`. It never calls live `submit_tick`, cannot omit or
override an encoded expected hash, and emits no live observation or
presentation work. After complete semantic success, but before the final
swap, Moria copies the exact verified source header to the selected fresh
stream as sequence zero and then copies every exact verified physical source
record—including every branch record—as sequences `1..=N`, one durable append
at a time. It does not flatten, regenerate, omit, or reorder those bytes. Only
after all `N + 1` appends are durable does one final `FrontierBundle` swap
publish the new world and `ReplayCompleted`; its `replay` has
`durable_through_sequence == N` and `next_sequence == N + 1`, so the first new
tick is ordered after the copied prefix. Intermediate and superseded roots
were never public. Cancellation, device loss, validation failure, mismatch,
sink failure, or wrong sink completion drains and drops all private state and
publishes no world. Failure before sequence-zero invocation releases the
stream reservations; failure/cancellation after it retires the pair, reports
the exact provider failure through `ReplayFailure.error`, and offers no
redrive or fallback.

Participant events are schema-bound opaque bytes in deterministic
`(ParticipantId, local_sequence)` order. Replay compares them exactly and
returns them in the corresponding replay tick receipt only for a published
replay frontier; Moria does not decode their behavior meaning, place them in
the Moria-state observation ring, or deliver them to another participant
during the same tick.

At the first mismatch, replay stops before advancing even the private replay
frontier past the divergent tick and returns
`ReplayFailure { error: ReplayDivergence, divergence: Some(...) }`. The
`moria-divergence-v1` value contains the fields above, including the exact
length-prefixed input prefix through the earliest tick. Admission rejects when
that prefix plus the worst-case changed-key/difference evidence cannot fit
`ReplayLimits.max_artifact_bytes` and
`rollback.divergence_artifact_bytes`; it never truncates the prefix or reports
a later tick. Non-divergence failures set `divergence: None`. The artifact
never calls a self-reported pass authoritative; an independent tool compares
bytes.

### TECH-048 — Rollback correction transaction

Implements: REQ-029, REQ-030, REQ-035, REQ-037, REQ-040, REQ-043

```rust
pub struct CorrectionRequest {
    pub world: WorldId,
    pub target: FrontierSummary,
    pub replacement_batches: BoundedVec<SealedTickBatch>,
    /// Empty, or exactly one expected post-tick root per replacement batch.
    pub expected_hashes: BoundedVec<CanonicalHash>,
    pub max_private_bytes: u64,
}

pub struct CorrectedTickOutput {
    pub tick: Tick,
    pub outcome_digest: CanonicalHash,
    pub participant_event_digest: CanonicalHash,
    pub participant_events: BoundedVec<ParticipantEvent>,
}

pub struct CorrectionCommitted {
    pub frontier: FrontierSummary,
    pub ticks: BoundedVec<CorrectedTickOutput>,
    pub replay: ReplayStreamPosition,
}
```

The TECH-070 `request_correction` call accepts only a retained
confirmed frontier strictly before the current confirmed frontier and a
complete contiguous input sequence from `target.tick + 1` through that current
frontier's tick. Thus v1 replaces a suffix without changing the numbered
present; shortening, extending, or submitting an empty correction is
`InvalidRequest`. The current physical replay stream must already be durable
through the current frontier with no append in flight; otherwise admission
returns `PersistenceBackpressure` and invokes no participant or sink. This
gives the branch one unambiguous next physical sequence and a fully durable
superseded suffix.

`expected_hashes` has exactly two legal cardinalities. Length zero disables
the optional consumer-supplied comparison. Otherwise its length must equal
`replacement_batches.len()`. Any nonzero short or excess vector is rejected
before pins, permits, private participant work, or sink invocation with
`AdmissionCode::CorrectionHashCountMismatch` and
`AdmissionContext::CorrectionExpectedHashCount { replacement_batches,
expected_hashes }` plus `Retryability::RetryNewRequest`;
`Rejected<CorrectionRequest>` returns both bounded vectors and every sealed
batch unchanged. Counts are exact `u32` values already bounded by
`rollback.ticks_per_correction`, so conversion or count overflow is also an
admission rejection rather than truncation. For a nonempty vector,
index `i` maps to `replacement_batches[i]` and to checked tick
`target.tick + 1 + i` in the same contiguous order. `expected_hashes[i]` is
compared byte-for-byte with that tick candidate's
`FrontierSummary.root_hash` before the private frontier advances. A mismatch
fails the accepted `CorrectionReceipt` with
`CorrectionError { original_frontier, error: OperationError {
code: ReplayDivergence, scope: Operation(receipt_id),
retryability: RetryNewRequest, committed: None, ... },
replay_export_failure: None }`; no branch is encoded or exported. An empty
vector disables only this optional comparison and never disables intrinsic
transition, participant-commitment, or branch-validation checks.

After those structural checks, admission reserves replay bytes, output roots,
participant resources, and pins the original live and target roots before
starting. It also computes the worst-case single encoded branch length and
rejects before private work unless the actual replacement count/bytes can fit
`rollback.ticks_per_correction`, `rollback.bytes_per_correction`,
`rollback.log_ticks`, `rollback.log_bytes`,
`rollback.replay_sink_bytes_in_flight`, and the configured
`ReplaySinkDescriptor.max_record_bytes`. Admission rejection returns the
complete request, including every still-owned sealed batch. Acceptance returns
`CorrectionReceipt`; no batch may be submitted independently while it belongs
to that correction.

Moria creates a private replay context from the target root and target
participant tokens. Snapshot adapters return new staged tokens from the pinned
target snapshot; reconstructible adapters return new staged tokens after
replaying the bounded prefix. Each replacement tick consumes only the prior
private token and returns the next one through the ordinary transition.
Intermediate roots and participant tokens remain private and do not emit
consumer observations or presentation work. Before any durable export,
failure or cancellation discards private roots and keeps the original live
frontier, rollback deque, active log, and replay stream position unchanged.

After every replacement tick validates, Moria encodes one bounded
`moria-replay-v1` correction-branch record. Its embedded frames are the exact
standalone tick-record bytes that ordinary confirmation would have produced,
including corrected outcomes, participant commitments/events, and root hashes.
The request is:

```text
stream = the world's frozen ReplayStreamKey
sequence = current ReplayStreamPosition.next_sequence
range = CorrectionBranch {
  target_tick,
  superseded_through = original_live_tick,
  corrected_through = original_live_tick,
  record_count = original_live_tick - target_tick
}
```

Moria reserves the immutable branch bytes, callback cell, and sink
count/byte permit before invocation. The branch is one atomically durable sink
record: `stored` means the exact complete record is visible at that sequence,
while `fail`, drop, or wrong completion means no record is visible there.
Cancellation is accepted only before this invocation. Once invoked,
`CorrectionReceipt::cancel` returns `NotCancellable`; a matching durable
completion makes corrected publication mandatory, and shutdown/device-loss
notifications are ordered after that publication. All candidate GPU work and
generation checks finish before invocation, so the remaining main-world
publication is an infallible host transaction.

On matching durability, the exclusive TECH-032 critical section atomically:

1. replaces rollback-deque entries after the target with the corrected
   frontiers;
2. splices the in-memory active log to
   `prefix_through_target || corrected_records`;
3. advances the physical `ReplayStreamPosition` by this one branch record and
   installs the corrected active-history digest;
4. swaps the live `FrontierBundle`, receipt result, participant tokens,
   revision metadata, and one success observation; and
5. schedules only the final accumulated dirty regions for derived rebuild.

Superseded roots, records, and participant tokens become reclaimable only
after their existing reader/checkpoint/query pins and GPU uses drain. Their
already-durable physical stream bytes are never deleted, reused, or considered
active log records after the branch. `CorrectionCommitted.frontier` is the
corrected live frontier and `CorrectionCommitted.replay` is the advanced
durable physical position. The success `CorrectionObservation` carries the
same `to` frontier and `replay`; a failure carries `to: None, replay: None`.

The correction observation contains Moria-owned frontier/outcome facts only.
Participant-owned events from private replay ticks are delivered in the
bounded `CorrectionCommitted.ticks` result after final publication and in the
embedded tick frames of the correction-branch record, never through TECH-025
or during private replay. Admission
reserves their worst-case aggregate count/bytes under the correction and
terminal-receipt budgets.

Original frontiers and participant tokens remain pinned until success/failure
and all GPU readers complete. On failure, staged CPU tokens drop immediately
after callback closure and staged GPU tokens enter generation-tagged retire
queues until their last submission completes; no participant restore-back call
is needed or permitted. Target outside the window, missing state,
resource bound, participant failure, content mismatch, or divergence before
branch invocation is terminal for that correction and never advances the
world. Every such receipt failure is
`CorrectionError { original_frontier, error, replay_export_failure: None }`
with `error.committed == CommittedEffect::None`; retryability follows the
underlying failure and participant policy. A branch append failure also leaves
the original bundle/log installed but applies TECH-047's terminal
provider-failure policy. Its receipt error is
`CorrectionError { original_frontier, error: OperationError {
code: StoreFailure, scope: Provider(ReplaySink(id)), retryability: Never,
committed: None, ... }, replay_export_failure: Some(the_exact_failure) }`.
Moria appends one `CorrectionObservation { from: original_frontier, to: None,
replay: None, failure: Some(StoreFailure) }` and one
`WorldLifecycleFact { state: Failed, frontier: original_frontier,
failure: Some(the_same_operation_error),
replay_export_failure: Some(the_exact_failure), ... }`; the last readable
frontier remains byte-identical to all three records. It retains the exact
undurable `ReplayExportFailure` and closes new authority admission; there is
no redrive or alternate sink. Because a successful append mandates the one
host publication transaction, no reachable state has a durable correction
branch while continuing to expose the superseded live frontier.

### TECH-049 — Replay/log and checkpoint bounds

Implements: REQ-015, REQ-018, REQ-021, REQ-029, REQ-032

The in-memory confirmed log is the active semantic projection, not the raw
physical append sequence. It retains at least the rollback window and at most
`rollback.log_ticks` (default 256), `rollback.log_bytes` (default 256 MiB), and
the newest durable checkpoint suffix. Each active entry stores its exact tick
record bytes/digest plus physical `(sequence, subrecord_offset)` locator.
Correction atomically splices this log after its target; superseded entries
leave the active count immediately but remain pinned by preexisting readers
until drain. The configured replay sink exports exact immutable physical
records under TECH-047. An ordinary record is releasable only after its exact
sink append is durable; a correction's embedded records become releasable
under the same rule only after the containing branch append is durable and
the branch publication transaction completes. Checkpoint coverage may release
other recovery pins but never substitutes for the public replay export. At
the boundary, `reserve_tick` returns `PersistenceBackpressure`; Moria never
silently drops the only recovery or requested replay record.

Checkpoint traversal defaults to 16 MiB mapped bytes and 64 MiB store writes
in flight, with three staging slots shared by scar, participant, and replay
blobs.
The sum of declared snapshot maxima must fit the configured checkpoint byte
budget and the 64 MiB per-frontier compiled maximum; otherwise genesis or
checkpoint admission fails before export. A manifest may reference at most the
configured scar nodes/bricks and 4 GiB uncompressed data in v1; lower consumer
limits are honored. Counts/offsets use checked `u64`, while individual wire
sequences remain `u32` bounded.

The union of reconstructible participant ranges must also fit
`rollback.log_ticks`, `rollback.log_bytes`, the checkpoint request's total byte budget,
the 4 GiB manifest maximum, and the configured three-slot mapped/store
in-flight byte limits.
Replay-record and replay-chunk bytes count in checkpoint progress, completion,
telemetry, manifest completeness totals, and recovery-anchor accounting. A
checkpoint is a recovery anchor for a reconstructible participant only after
every required chunk and then the manifest are durable. Once that anchor is
visible, older in-memory records may be released only if they are not required
by rollback, correction, another participant range, or an attached replay
sink.

Rollback capacity is tick- and byte-bounded. A tick whose worst-case COW state
would exceed the canonical budget receives
`FailedNoAdvance { cause: TickNoAdvanceCause::Canonical(
CanonicalFailure::LogicalCapacity), error.code:
ErrorCode::CanonicalBudget, ... }`;
runtime pressure does not evict the required 20 confirmed frontiers.
Public replay request records, correction branch bytes and embedded record
index, private roots/results, sink completion records, and divergence artifact
bytes count against the dedicated TECH-017 rollback fields; no replay path
borrows untracked checkpoint or terminal-receipt memory. A branch whose single
physical record cannot fit the configured sink/byte permits is rejected before
private correction begins, never split into a partially durable transaction.

## Reclamation and dirty truth

### TECH-050 — Dirty tracking and safe eviction

Implements: REQ-004, REQ-014, REQ-015, REQ-018

Each confirmed root carries a persistent dirty-key set relative to the newest
durable checkpoint. It is updated by stable merge with changed scar and
metadata keys. Checkpoint success advances the durable root and computes the
remaining dirty set from later confirmed roots; it does not clear later keys.

A canonical node/brick is reclaimable only when absent from:

- the live root;
- every retained rollback frontier;
- the newest durable/recovery root still being read;
- active replay/correction/query/checkpoint root pins;
- participant artifact leases;
- submitted GPU work.

A resident base-cache brick additionally requires no interest/admitted-use
pin. Presentation can be discarded independently. Unpersisted scars are not
lost when their detailed resident brick retires because their immutable scar
leaf remains in all required roots. Shutdown reports every dirty root if final
persistence fails.
