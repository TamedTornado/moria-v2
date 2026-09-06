# Collision, participant interop, and presentation

Collision is a canonical derivation of material truth. Presentation is a
discardable consumer of that truth. They share source roots but never
authority.

## Canonical occupancy and collision

### TECH-051 — Collision primitive set

Implements: REQ-010, REQ-019, REQ-028, REQ-044

`CollisionShapeQ` is the closed set:

```rust
pub enum CollisionShapeQ {
    Point { point: WorldPointQ },
    Aabb { center: WorldPointQ, half_extent: WorldVectorQ },
    Sphere { center: WorldPointQ, radius: PlacementScalar },
    Capsule {
        a: WorldPointQ,
        b: WorldPointQ,
        radius: PlacementScalar,
    },
    OrientedBox {
        center: WorldPointQ,
        half_extent: WorldVectorQ,
        orientation: QuatQ14,
    },
}
```

Extents/radii are nonnegative, a capsule may degenerate to a sphere, and all
shapes must fit the request bounds. `Trace` uses a closed fixed-point segment.
`Sweep` linearly translates a shape without rotating it during the sweep;
consumers represent rotation as separate bounded steps. These primitives cover
point/region occupancy, traces, overlaps, and swept movement without exposing
meshes. More primitives require a new versioned collision contract.

Each occupied local cell is its exact closed-open cell box transformed by the
volume's canonical placement. Broad phase compares checked world AABBs.
Narrow phase transforms the consumer shape into volume-local space and uses
fixed-point slab, closest-feature, and separating-axis tests. Products and
squared distances use TECH-071's exact 64-bit semantics under the source
world's frozen placement split. Contact normal is a
canonical signed Q1.14 vector; ties select x, then y, then z, then negative
before positive face. `TimeOfImpactWire` is an unsigned Q0.32 value stored in
`u64` with the only valid range `0..=0x1_0000_0000`, so exact time one is
representable. Conversion uses floor rounding, conservatively choosing the
earliest representable contact.

The normative collision algorithm is `moria-collision-v1`:

1. A placement maps local point `p` by TECH-007's exact rational-rotation
   `translation + R(p - pivot)` sequence. World shapes are converted to volume
   local space with the exact inverse sequence. Sphere radius and box/capsule
   extents are unchanged; AABB orientation becomes `inverse(volume_q)`,
   oriented-box orientation becomes
   `normalize(inverse(volume_q) * shape_q)`, and capsule endpoints are
   transformed separately. No world-space float AABB is fed to narrow phase.
2. Let `E = PlacementFixedFormat::cell_extent_raw()`. Cell `(x,y,z)` has raw
   bounds
   `[E*x,E*(x+1)) × [E*y,E*(y+1)) × [E*z,E*(z+1))`, with every product
   checked in `i64` and required to fit `i32`. A point overlaps only when
   `lo <= p < hi` on every axis. For positive-size shape contact,
   the geometric high planes are included and equality is overlap; duplicate
   boundary cells remain distinct facts and TECH-024 orders them.
3. Sphere/box uses `q_i = clamp(c_i, lo_i, hi_i)`,
   `d = c - q`, and overlap iff `dot(d,d) <= r*r`. If `d == 0`, the witness
   face is the minimum of `(c-lo, hi-c)` in axis/sign tie order. Capsule/box
   minimizes squared distance from `p(u)=a+u(b-a)`, `0<=u<=1`, to the box.
   It creates rational breakpoints `(plane-a_i)/(b_i-a_i)` for every crossed
   low/high plane, sorts them by checked cross multiplication, and on each
   interval fixes the three clamp states. The resulting
   `D(u)=A*u*u+B*u+C` is evaluated at both endpoints and at
   `clamp(-B/(2A))` when `A != 0`; rational values are compared without first
   rounding. The winning witness is the least `u`, then axis/sign order.
   Capsule overlap is `D_min <= r*r`. A zero-length capsule takes the sphere
   path; a zero-radius sphere/capsule takes the point/segment path.
4. AABB/box and oriented-box/box use SAT. Shape vertices are made once by
   applying TECH-007 rotation to each signed half-extent and adding the center.
   The ordered axes are cell x/y/z, shape x/y/z, then
   `cross(cell_axis_i, shape_axis_j)` for `(i,j)` lexicographically. Zero axes
   are skipped. Vertex projections are checked `i128` dot products. Strictly
   disjoint intervals separate; touching intervals overlap. The contact axis
   has minimum rational depth `interval_overlap / ceil_sqrt(dot(axis,axis))`;
   depths compare by cross multiplication, then axis-list order. Its sign
   points from the cell center toward the shape center; a zero center
   projection selects the negative sign before positive.
5. Point/box trace uses the three slab intervals. For axis delta `d==0`, a
   start outside that slab rejects; otherwise entry and exit are the exact
   plane fractions `(plane-start)/d`, with numerator/denominator sign
   normalized. Fractions are intersected with `[0,1]` by `i128` cross
   multiplication in x/y/z order. AABB and oriented-box sweeps use continuous
   SAT on the same ordered axes: with projected velocity `v`, each axis
   contributes entry `(cell_min-shape_max)/v` and exit
   `(cell_max-shape_min)/v`, swapped when `v<0`; `v==0` must already overlap.
6. Translating sphere/box is the same piecewise quadratic closest-feature
   calculation as step 3 with tick parameter `t`. Translating capsule/box
   minimizes `distance(box, a + t*delta + u*(b-a))` over
   `[t,u] in [0,1]^2`. For each of the 27 coordinate clamp-state combinations,
   it solves the checked 2×2 integer normal equations, evaluates every feasible
   interior solution and all `t/u` boundaries, and rejects candidates outside
   their clamp region. Singular systems reduce to their one-dimensional
   boundary cases. Candidates are ordered by earliest `t`, then least `u`,
   then clamp-state axis/sign order. This finite active-set enumeration is the
   only capsule-sweep algorithm; iterative conservative advancement is not
   conforming.
7. Linear and quadratic crossing fractions remain exact until output.
   Quadratic discriminants use checked `i128` and unsigned integer square root.
   The earliest crossing is converted with
   `floor(2^32 * numerator / denominator)`; an irrational quadratic root uses
   the greatest Q0.32 integer whose rational time does not exceed the root,
   found by a fixed 32-step high-to-low bit test against the original
   polynomial. Initial overlap is time zero. If the interval intersection is
   empty through time one, the result is `NoHit`.
8. A nonzero local witness/axis `n` becomes Q1.14 using TECH-007's exact
   square-root rounding comparison with scale 16,384 and no sign
   canonicalization; its direction is from the occupied cell toward the query
   shape. An interior zero witness uses the selected face unit vector.
   Sphere/capsule local contact point is the box witness; polytope local
   contact point is the component-wise ties-to-even midpoint of the cell
   support point along `n` and shape support point along `-n`, with
   support-vertex ties resolved by encoded vertex order. Sweep witnesses are
   evaluated at the exact winning rational time and only then rounded once to
   local placement raw units with TECH-071 ties-to-even.
9. Public contact facts are always world-space. Transform the local contact
   point with TECH-007's exact `translation + R(local - pivot)` sequence.
   Rotate the directed local Q1.14 normal with TECH-007's same rational
   numerator/denominator, reducing each component once to Q1.14 ties-to-even,
   then normalize that rounded vector with the exact square-root comparison
   from TECH-007 and without quaternion-style sign canonicalization. A zero
   rotated vector or failed quantized-unit-shell postcondition is an arithmetic
   failure. This conversion occurs after winner selection and cannot change
   fact ordering or the selected face.

Every fraction has a positive denominator and is reduced by binary GCD before
storage. No epsilon exists. A zero sweep delta runs static overlap and returns
zero or `NoHit`. Invalid extents, a nonrepresentable transform, checked
overflow, singular case not covered by the stated boundary reduction, or a
world normal/contact outside its wire range is a typed collision arithmetic
failure;
ordinary queries become `Unavailable`, while a canonical participant tick is
`FailedNoAdvance`. CPU and WGSL must execute the same axis, candidate, and
reduction order.

### TECH-052 — Sparse collision traversal and facts

Implements: REQ-001, REQ-010, REQ-017, REQ-019, REQ-021

Collision reads the pinned canonical root and verified resident base bricks,
never mesh, dressing, or presentation bounds. It traverses volumes in
`VolumeId` order, radix brick ranges in Morton-independent lexicographic
`(z,y,x)` order, then cells. Uniform empty nodes reject whole regions. Exact
brick occupancy masks are derived from `CellWire` and cached with their source
leaf hash; masks may accelerate but cannot report occupied matter empty.

Trace uses three-axis integer DDA with rational comparisons; sweep broad phase
uses the union of start/end bounds. Candidate facts occupy fixed input slots,
then stable mark/scan/scatter and sort establish TECH-024 order. A complete
fact uses the following named wire types:

```rust
pub struct WorldContactPointQWire(pub [i32; 3]); // world's placement raw units
pub struct WorldContactNormalQWire(pub [i16; 3]); // Q1.14, cell toward shape
pub struct TimeOfImpactWire(pub u64); // Q0.32, 0..=0x1_0000_0000

pub struct CollisionFact {
    pub source_frontier: FrontierPosition,
    pub world_root: CanonicalHash,
    pub volume: VolumeId,
    pub revision: VolumeRevision,
    pub local_cell: LocalCellPoint,
    pub material: MaterialId,
    pub time_of_impact: TimeOfImpactWire,
    pub world_contact_point: WorldContactPointQWire,
    pub world_contact_normal: WorldContactNormalQWire,
    pub source_leaf_hash: CanonicalHash,
}
```

The normal is not sign-canonicalized because its direction is semantic. The
closed `CollisionFact` above is the complete fact; its field order is its wire
order. Conceptually it contains:

```text
source frontier, world root, volume/revision, local cell, material,
time_of_impact, world_contact_point: WorldContactPointQWire,
world_contact_normal: WorldContactNormalQWire, source leaf hash
```

`source_frontier` uses TECH-070's exact canonical
`0 = Genesis | 1 = Confirmed(Tick)` encoding. It identifies the pinned state
from which the fact was derived: a query against the pre-tick root emits
`Genesis`, while collision for attempted tick `n` emits the position of
`SourceState(n)`. `Genesis` and `Confirmed(Tick::from_raw(0))` therefore have
different fact bytes and cannot be substituted even when all other fields
match.

`NoHit` is emitted only if every required brick was ready and inspected.
Missing/cold/corrupt truth is `Pending` or `Unavailable`. Canonical participant
collision for attempted tick `n` is prepared against `SourceState(n)`; absent
or source-mismatched input
causes tick `FailedNoAdvance`, never collision-disabled success.

### TECH-053 — Canonical collider artifact

Implements: REQ-005, REQ-006, REQ-019, REQ-030

A participant descriptor may reserve collider artifacts by bounded regions,
shape classes, maximum occupied cells, and bytes. Moria builds a canonical
artifact from a pinned state:

```text
header {
  contract, source_frontier, world_root, request_digest,
  volume_count, record_count, complete
}
sorted volume records { id, revision, inverse placement, range }
sorted occupied spans { volume_id, brick_coord, z/y/x run, material, density }
artifact_hash
```

Spans are a deterministic lossless encoding of occupancy; capacity overflow is
an unavailable artifact, not truncation. The artifact hash binds all bytes and
source roots. CPU participants receive decoded immutable bytes. GPU
participants receive the same bytes in a read-only artifact buffer. This is a
bounded derived representation, not a second authority: it is accepted only
for its exact source hash and can always be rebuilt from matter.

## GPU behavior participant seam

### TECH-054 — Bounded GPU participant adapter

Implements: REQ-002, REQ-005, REQ-006, REQ-012, REQ-030, REQ-044

The deliberately Bevy/wgpu-coupled seam is:

```rust
pub struct GpuParticipantDescriptor {
    pub participant: ParticipantDescriptor,
    pub shader_contract: ContractDigest,
    pub workgroup_size: u32,
    pub scratch_bytes: u64,
}
pub struct GpuParticipantContract {
    pub device_generation: DeviceGeneration,
    pub granted_limits_digest: ContractDigest,
    pub io_abi: ContractDigest,
    pub collider_abi: ContractDigest,
    pub effect_abi: ContractDigest,
    pub event_abi: ContractDigest,
    pub state_abi: ContractDigest,
}
impl GpuParticipantContract {
    pub fn io_bind_group_layout(&self) -> &wgpu::BindGroupLayout;
}
pub struct GpuBufferRange {
    pub offset: u32,
    pub bytes: u32,
}
pub enum GpuParticipantOperation {
    Genesis,
    Tick,
    RestoreSnapshot,
    Reconstruct,
    ExportSnapshot,
}
pub struct GpuParticipantIoMetadata {
    pub operation: GpuParticipantOperation,
    pub participant: ParticipantId,
    pub world: WorldId,
    pub source_frontier: Option<FrontierPosition>,
    pub destination_frontier: Option<FrontierPosition>,
    pub attempted_tick: Option<Tick>,
    pub source_root: CanonicalHash,
    pub source_commitment: CanonicalHash,
    pub artifact_hash: CanonicalHash,
    pub input: GpuBufferRange,
    pub collider: GpuBufferRange,
    pub source_state: GpuBufferRange,
    pub destination_state: GpuBufferRange,
    pub effects: GpuBufferRange,
    pub events: GpuBufferRange,
    pub effect_capacity: u32,
    pub event_capacity: u32,
    pub device_generation: DeviceGeneration,
}
pub struct GpuParticipantStateLease<'a> { /* borrowed immutable state binding */ }
pub struct ParticipantGpuInput<'a> { /* borrowed input/artifact bindings */ }
pub struct ParticipantEffectSink<'a> { /* borrowed fixed-slot effect binding */ }
pub struct ParticipantEventSink<'a> { /* borrowed fixed-slot event binding */ }
pub struct GpuParticipantStateSink<'a> { /* borrowed unreferenced state binding */ }
pub struct GpuSnapshotInput<'a> { /* borrowed verified snapshot binding */ }
pub struct GpuSnapshotOutput<'a> { /* borrowed fixed-size staging binding */ }
pub struct GpuParticipantReplayInput<'a> { /* borrowed exact replay binding */ }

impl<'a> GpuParticipantStateLease<'a> {
    pub fn metadata(&self) -> ParticipantTokenMetadata;
    pub fn range(&self) -> GpuBufferRange;
}
impl<'a> ParticipantGpuInput<'a> {
    pub fn bind_io(&self, pass: &mut wgpu::ComputePass<'_>);
    pub fn metadata(&self) -> GpuParticipantIoMetadata;
    pub fn input_range(&self) -> GpuBufferRange;
    pub fn collider_range(&self) -> GpuBufferRange;
}
impl<'a> ParticipantEffectSink<'a> {
    pub fn range(&self) -> GpuBufferRange;
    pub fn capacity(&self) -> u32;
}
impl<'a> ParticipantEventSink<'a> {
    pub fn range(&self) -> GpuBufferRange;
    pub fn capacity(&self) -> u32;
}
impl<'a> GpuParticipantStateSink<'a> {
    pub fn bind_io(&self, pass: &mut wgpu::ComputePass<'_>);
    pub fn metadata(&self) -> GpuParticipantIoMetadata;
    pub fn range(&self) -> GpuBufferRange;
}
impl<'a> GpuSnapshotInput<'a> {
    pub fn bind_io(&self, pass: &mut wgpu::ComputePass<'_>);
    pub fn metadata(&self) -> GpuParticipantIoMetadata;
    pub fn bytes(&self) -> GpuBufferRange;
    pub fn digest(&self) -> BlobDigest;
}
impl<'a> GpuSnapshotOutput<'a> {
    pub fn bind_io(&self, pass: &mut wgpu::ComputePass<'_>);
    pub fn metadata(&self) -> GpuParticipantIoMetadata;
    pub fn range(&self) -> GpuBufferRange;
    pub fn capacity_bytes(&self) -> u32;
}
impl<'a> GpuParticipantReplayInput<'a> {
    pub fn bind_io(&self, pass: &mut wgpu::ComputePass<'_>);
    pub fn metadata(&self) -> GpuParticipantIoMetadata;
    pub fn range(&self) -> GpuBufferRange;
    pub fn first_tick(&self) -> Option<Tick>;
    pub fn last_tick(&self) -> Option<Tick>;
    pub fn record_count(&self) -> u32;
    pub fn digest(&self) -> BlobDigest;
}

pub trait BevyGpuParticipant: Send + Sync + 'static {
    fn descriptor(&self) -> GpuParticipantDescriptor;
    fn prepare_device(
        &self,
        device: &RenderDevice,
        contract: &GpuParticipantContract,
    ) -> Result<Box<dyn GpuParticipantDeviceState>, ParticipantError>;
}

pub trait GpuParticipantDeviceState: Send + Sync {
    fn encode_genesis(
        &mut self,
        pass: &mut wgpu::ComputePass<'_>,
        output: GpuParticipantStateSink<'_>,
    ) -> Result<(), ParticipantError>;
    fn encode_tick(
        &mut self,
        pass: &mut wgpu::ComputePass<'_>,
        source: GpuParticipantStateLease<'_>,
        input: ParticipantGpuInput<'_>,
        effects: ParticipantEffectSink<'_>,
        events: ParticipantEventSink<'_>,
        output: GpuParticipantStateSink<'_>,
    ) -> Result<(), ParticipantError>;
    fn encode_restore_snapshot(
        &mut self,
        pass: &mut wgpu::ComputePass<'_>,
        snapshot: GpuSnapshotInput<'_>,
        output: GpuParticipantStateSink<'_>,
    ) -> Result<(), ParticipantError>;
    fn encode_reconstruct(
        &mut self,
        pass: &mut wgpu::ComputePass<'_>,
        request: GpuParticipantReplayInput<'_>,
        output: GpuParticipantStateSink<'_>,
    ) -> Result<(), ParticipantError>;
    fn encode_snapshot_export(
        &mut self,
        pass: &mut wgpu::ComputePass<'_>,
        source: GpuParticipantStateLease<'_>,
        output: GpuSnapshotOutput<'_>,
    ) -> Result<(), ParticipantError>;
}
```

The `io_abi` fixes one Moria-created group-zero layout for every operation.
Binding 0 is a 224-byte uniform `moria-participant-io-v1` wire; bindings 1
through 6 are respectively operand input/snapshot/replay, collider artifact,
source state, destination state/status, effect output/status, and event
output/status. The uniform byte offsets are: operation tag `0`, participant
`4`, 16-byte world ID `8`, source-position tag/low/high `u32` words
`24/28/32`, destination-position tag/low/high words `36/40/44`,
attempted-tick presence/low/high words `48/52/56`, one reserved zero word at
`60`, device-generation low/high words `64/68`, effect/event capacities
`72/76`, source root `80..112`, source commitment `112..144`, artifact hash
`144..176`, and six `{offset:u32, bytes:u32}` ranges in binding order at
`176..224`. Optional position tag `0` means absent, tag `1` means `Genesis`
and requires zero tick words, and tag `2` means `Confirmed` followed by its
exact tick; attempted-tick presence is `0` or `1` and absent tick words are
zero. Every scalar is little-endian, reserved bytes must be zero, and every
unknown tag is invalid.
`GpuParticipantIoMetadata` is the decoded host view of those bytes, not a Rust
struct transmute.

The operation-specific position contract is closed:

| Operation | Source frontier | Destination frontier | Attempted tick |
| --- | --- | --- | --- |
| `Genesis` | `None` | `Some(Genesis)` | `None` |
| `Tick` | `Some(SourceState(n).position)` | `Some(Confirmed(n))` | `Some(n)` |
| `RestoreSnapshot` | `Some(snapshot.frontier.position)` | the same position | `None` |
| `Reconstruct` | `Some(request.start.position)` | `Some(Confirmed(request.end_tick))` | `None` |
| `ExportSnapshot` | `Some(source_token.frontier.position)` | `None` | `None` |

For `Tick`, `source_frontier.next_tick()` must equal `attempted_tick`; for
restore the two positions must be byte-identical. Genesis output metadata and
the installed token both carry `Genesis`, never zero-filled tick words
interpreted as `Confirmed(0)`. When `source_frontier` is absent,
`source_root` and `source_commitment` are all-zero non-identity wire values;
when present they must match the source token/frontier. `artifact_hash` is
nonzero only for an operation carrying a collider artifact. All
operation-unused ranges, capacities, hashes, and option payload words are
zero, so no stale tick/frontier data survives wrapper reuse.

An unused binding references a shared four-byte zero buffer with a logical
range of zero. Each storage range is page-local, fits `u32`, lies inside its
effective binding, and its physical binding offset is aligned to the granted
`min_storage_buffer_offset_alignment`; larger world pools remain paged under
TECH-033. `prepare_device` receives the exact `io_bind_group_layout` and the
adapter is required to include it as group zero in every participant pipeline.
wgpu does not expose pipeline-layout or encoder identity from a
`ComputePass`, so this requirement is not claimed to be eagerly inspectable by
a wrapper.

Exactly one wrapper is the primary binder for an encode call:
`GpuParticipantStateSink` for genesis, `ParticipantGpuInput` for tick,
`GpuSnapshotInput` for restore, `GpuParticipantReplayInput` for reconstruct,
and `GpuSnapshotOutput` for export. Its `bind_io` sets only group zero with no
dynamic offsets. Every other wrapper in that call is a range/capacity view
into the same generation/attempt bind group. Moria verifies that all wrappers
share the same private attempt token before invoking the adapter; wrappers
cannot be constructed, mixed, retained past `'a`, or rebound by consumer code.

`ParticipantGpuInput` thereby exposes the bounded canonical collider artifact,
immutable participant input bytes, exact counts, source/destination frontier,
attempted tick, root, and artifact hash,
leased source state, and declared scratch ranges without returning Moria's
radix, brick, page-table, allocator, or mutable buffers.
`ParticipantEffectSink` is a Moria-owned fixed-slot range whose schema is
ordinary command wire plus overflow/count status. The parallel
`ParticipantEventSink` is a fixed-slot range of `ParticipantEvent` headers,
opaque bytes, and status. The destination-state header contains completion,
state length, commitment, RNG-state digests, and snapshot metadata. All
capacities, byte ranges, overflow flags, and zeroed unused slots are fixed
before encoding; the participant cannot submit the encoder, bind a consumer
buffer, access a range outside the wrappers, or increase capacity.

`prepare_device` owns only pipelines, layouts, and other rebuildable
generation resources; it may not contain the participant's active canonical
state. `GpuParticipantStateSink` allocates from the participant's declared
fixed-capacity state pool and returns an immutable generation-tagged opaque
token. Tick, restore, and reconstruct encoders read one source and write one
unreferenced destination; source and destination aliasing is rejected before
encoding. Moria pins source/destination buffers through queue completion and
then installs or aborts tokens solely through the `FrontierBundle` lifecycle
in TECH-016. Snapshot export copies exactly the descriptor-bounded bytes to a
Moria staging slot; mapping, digest verification, and `CheckpointStore`
handoff follow TECH-045. It is never same-frame CPU visibility.

Every accessor is allocation-free and valid only for the borrowed
device-generation lifetime. `bind_io` is an infallible call to
`ComputePass::set_bind_group(0, ...)` for the wrapper's Moria-created group;
it does not and cannot query the pass's selected pipeline, later pipeline
changes, encoder identity, layout compatibility, or device generation.
Moria guarantees wrapper attempt/generation consistency when it constructs
the borrowed call arguments. The complete participant encoding is enclosed in
one balanced wgpu validation scope that is popped and resolved before the
attempt may publish. Selecting an incompatible pipeline before or after
`bind_io`, dispatching with a mismatched group-zero layout, or otherwise
misusing the pass therefore yields scoped wgpu validation, fails the owning
participant attempt as `ParticipantFailure`, and publishes no destination
token, effects, events, or canonical frontier. An adapter's own synchronous
validation may still make `encode_*` return `Err`, but `bind_io` itself has no
fallible milestone.
Generation loss closes every wrapper from that generation, rejects any
mapped/status result as `LateGeneration`, and retains its ranges only until
the last submitted use completes. Moria maps or copies back the bounded status,
effect, event, destination-state, and snapshot ranges through the ordinary
TECH-032 path; missing completion status, count/byte overflow, malformed
records, unexpected writes in unused ranges, or commitment mismatch fails the
attempt before publication. No output is inferred merely because
`encode_*` returned `Ok`.

Moria invokes adapters in `ParticipantId` order in the canonical preparation
schedule, then validates effect tags, bounds, preconditions, unique local
sequences, event schema/payload bounds, unused-zero slots, overflow flags,
participant and RNG-state commitments, state-token metadata, and snapshot
digest/size before normal transition processing. The output remains
GPU-resident through validation and application; only bounded canonical
outcomes, opaque event bytes, and commitments are read back for receipt,
and replay. Thus a GPU behavior engine need not round-trip occupancy or effects
through CPU, while admission and publication remain identical.

The GPU adapter obeys TECH-029's same one-phase simplification: it receives no
other participant's destination state, effects, or events; the event sink is
consumer delivery after confirmation, not a same-tick handoff; and effect
conflicts use TECH-011's ordinary phase-4 ordering. No render-graph edge may
turn these outputs into a participant DAG or prior-feedback ABI.

Adapter pipeline/device creation failure, panic caught at the FFI/callback
boundary where possible, validation error, output overflow, or old generation
causes `FailedNoAdvance`, after which the descriptor's TECH-029
`ParticipantFailurePolicy` selects retryable last-frontier or terminal-world
handling. There is no automatic CPU implementation swap. Device resources
reconstruct in `RenderStartup`; old-generation state tokens are terminal and
are restored into new staged tokens from a durable snapshot or exact canonical
replay-record bytes before publication. Under
`NoAdvanceExplicitRetry`, generation loss holds the world in
`RecoveringParticipant` until that equality-checked reconstruction succeeds;
under `FailWorld` it enters `Failed`. A correction never asks the device
adapter to mutate its current token in place. On correction failure, shutdown,
or generation loss, uninstalled tokens drain their last queue use and return
to the bounded pool without affecting the pinned live token.

## Derived presentation

### TECH-055 — Hybrid surface derivation

Implements: REQ-001, REQ-003, REQ-013, REQ-020

Presentation is brick-local with a one-cell neighbor halo:

- `SmoothDensity` materials use surface nets over the signed Q8.8 density
  field. One vertex is produced per sign-changing cell by a deterministic
  integer average, then converted to float for rendering.
- `CrispCell` materials use greedy axis-aligned quads within a volume's local
  lattice. Merging depends only on material/presentation key and source face.
- Boundaries between styles emit the crisp material face plus a clipped smooth
  edge; duplicate faces resolve by stable material ID.

The baseline produces indexed Bevy meshes per `(volume, brick presentation
chunk, source revision)`. Dynamic volume vertices remain local and the Bevy
transform derives from canonical placement. Movement therefore changes the
transform, not material cells. Neighbor edits dirty the changed brick and its
26 halo neighbors so cuts and seams rebuild honestly.

Mesh vertex/index order and float values are noncanonical. Buffers and output
counts are bounded; overflow emits `PresentationStatus::Failed` for that
chunk through its `PresentationFact` and leaves matter/collision intact. Raw
cell display is a diagnostic option,
not the normal surface contract.

### TECH-056 — Presentation lifecycle and revision installation

Implements: REQ-011, REQ-012, REQ-013, REQ-017, REQ-021, REQ-040

Each presentation chunk follows:

```text
Absent -> Building(source root/revision)
       -> Current(source root/revision)
       -> Stale(old source) -> Building(new source)
       -> Failed(source, reason)
```

A job pins its source root. Installation compares world, volume, brick,
revision, root hash, and device generation with the current requested source.
A mismatch discards the result; it cannot become current. Consumers configure
whether stale meshes remain visible, disappear, or are replaced by a
diagnostic marker. Collision never reads any choice.

Dirty revisions coalesce to the newest committed source when no job is
submitted; a submitted job drains and is then discarded if stale. Rollback
cancels/invalidates future-source jobs and enqueues only the final corrected
dirty union. `PresentationCurrent` observations emit only after GPU mesh upload
and Bevy entity installation for the named revision.

### TECH-057 — Revision-anchored dressing and assemblies

Implements: REQ-003, REQ-013, REQ-020

Derived dressing is generated from a presentation surface plus a consumer
presentation seed that is explicitly noncanonical. Every dressing instance
carries supporting `(volume, brick, material, source leaf hash, revision)`.
When support changes or disappears, the instance is removed or regenerated
before the chunk can report current. Dressing has no collision, occupancy,
persistence, query identity, or canonical hash contribution.

Anything requiring independent material identity or occupancy is not dressing.
It is registered by the consumer as an ordinary static/dynamic material volume
or included in base/patch content, and therefore obeys normal IDs, ticks,
mutation, collision, rollback, and persistence. Moria supplies no vegetation,
clutter, geology, or assembly recipe.

### TECH-058 — Presentation resource and failure isolation

Implements: REQ-007, REQ-013, REQ-018, REQ-021, REQ-022

Presentation has a separate byte budget and at most three in-flight jobs by
default. Under pressure it coalesces dirty work, retires out-of-interest
chunks, keeps stale views per policy, or marks a chunk failed. It cannot claim
canonical pool permits, evict pinned truth, block a tick on shader readiness,
or feed mesh counts/timing into simulation.

Presentation telemetry reports requested/current/stale/failed chunks,
truth-to-view revision lag, mesh/vertex/index bytes, queue age, rebuild cause,
overflow, and commit-to-current latency. Rendering correctness is human/visual
evidence; presentation failure injection must leave identical canonical hashes
and collision facts.
