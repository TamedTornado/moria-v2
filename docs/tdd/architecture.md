# Canonical architecture and data model

This document defines state that can affect future authoritative results and
the mechanisms that publish it. Public Rust shapes are in
[interfaces.md](interfaces.md); concrete GPU scheduling is in
[gpu-runtime.md](gpu-runtime.md).

## Canonical identity and coordinates

### TECH-005 — Stable identity model

Implements: REQ-003, REQ-017, REQ-028, REQ-033

`WorldId`, `MaterialId`, `VolumeId`, `ParticipantId`, `InputSourceId`, and
`RngStreamId` are fixed-width newtypes. `WorldId` is a consumer-supplied 128-bit
value committed at genesis. Material IDs are consumer-supplied nonzero `u16`
values. Participant IDs are nonzero `u32` values no greater than
`0x7fff_ffff`; input-source IDs are nonzero `u32` values with their high bit
clear. RNG-stream IDs are nonzero `u32` values scoped to and unique within one
participant descriptor; Moria has no global RNG registry and consumes no
randomness of its own. Each other ID is unique in its world registry.
`VolumeId` is a nonzero `u64`. Genesis volumes may claim explicit unique IDs;
canonical `next_volume_serial` starts one above their maximum. Sorted
post-genesis create commands allocate and increment it. IDs are never reused,
including after retirement or rollback-window reclamation.

Duplicate genesis IDs, zero reserved IDs, exhausted counters, and a reference
to an absent or retired ID are typed validation failures. Physical node,
buffer, slot, entity, task, and submission IDs never appear in canonical
encoding, persistence, replay, observations, or the public identity types.

All identity and digest fields are private. The following are the exact
consumer construction and inspection methods; tuple construction is not part
of the public API:

```rust
pub enum NewtypeValueError {
    ZeroReserved,
    OutOfRange,
    AllZeroReserved,
}

pub struct WorldId([u8; 16]);
pub struct MaterialId(u16);
pub struct VolumeId(u64);
pub struct ParticipantId(u32);
pub struct InputSourceId(u32);
pub struct RngStreamId(u32);
pub struct Tick(u64);
pub struct VolumeRevision(u64);
pub struct CanonicalOrder(u32);
pub struct DeviceGeneration(u64);
pub struct ReceiptId(u64);

pub struct CanonicalHash([u8; 32]);
pub struct ContentDigest([u8; 32]);
pub struct ContractDigest([u8; 32]);
pub struct SchemaDigest([u8; 32]);
pub struct BlobDigest([u8; 32]);

impl MaterialId {
    pub fn try_from_raw(raw: u16) -> Result<Self, NewtypeValueError>;
    pub fn get(self) -> u16;
}
impl VolumeId {
    pub fn try_from_raw(raw: u64) -> Result<Self, NewtypeValueError>;
    pub fn get(self) -> u64;
}
impl ParticipantId {
    pub fn try_from_raw(raw: u32) -> Result<Self, NewtypeValueError>;
    pub fn get(self) -> u32;
}
impl InputSourceId {
    pub fn try_from_raw(raw: u32) -> Result<Self, NewtypeValueError>;
    pub fn get(self) -> u32;
}
impl RngStreamId {
    pub fn try_from_raw(raw: u32) -> Result<Self, NewtypeValueError>;
    pub fn get(self) -> u32;
}
```

`MaterialId::try_from_raw` and `VolumeId::try_from_raw` reject zero.
`ParticipantId` and `InputSourceId` reject zero and values greater than
`0x7fff_ffff`. `RngStreamId` rejects only zero: every value in
`1..=u32::MAX`, including `0x8000_0000`, is valid because the ID is scoped
inside one participant descriptor and never enters the shared participant /
input-source high-bit ordering namespace. There is no unchecked or lossy
constructor.

`WorldId` has `from_bytes([u8; 16])`, `as_bytes(&self) -> &[u8; 16]`, and
`to_bytes(self) -> [u8; 16]`; every 16-byte value is valid. `Tick`,
`VolumeRevision`, `CanonicalOrder`, `DeviceGeneration`, and `ReceiptId` have
infallible `from_raw` and `get` methods using their displayed scalar width;
zero is valid for those value/counter types. Each 32-byte hash/digest above has
infallible `from_bytes([u8; 32])`, `as_bytes(&self) -> &[u8; 32]`, and
`to_bytes(self) -> [u8; 32]`. Digest types are not implicitly interchangeable
even though their wire widths match. All types in this block are
`Copy + Eq + Ord + Hash`; constructors and accessors allocate nothing and
preserve every bit.

### TECH-006 — Material cells, bricks, and logical domains

Implements: REQ-001, REQ-003, REQ-004, REQ-020, REQ-028

The addressable unit is a cubic cell on a volume-local integer lattice. One
canonical `CellWire` is four bytes:

```text
offset  size  field
0       2     material_id: u16 little-endian
2       2     density_q8_8: i16 little-endian
```

`material_id == 0` is empty and requires `density_q8_8 <= 0`.
`material_id != 0` requires a registered material. Occupancy is determined by
that material's immutable genesis `OccupancyClass` and density threshold;
`Never` material remains inspectable matter but contributes no collision.
Density supplies signed coverage for honest interpolated boundaries; material
presentation style does not change occupancy. Invalid combinations are
rejected at content or patch validation rather than normalized silently.

A brick contains 8×8×8 cells in x-major, then y, then z order and has an exact
2,048-byte canonical payload. A brick may instead be represented by the same
four-byte uniform cell. Floor division and remainder for negative cell
coordinates are explicitly Euclidean: `q = floor(a/8)`, `r = a - 8q`,
`0 <= r < 8`.

A volume domain is a nonempty, half-open local-cell AABB whose coordinates are
`i32`, each side is at most 8,191 cells, and every corner is within 4,095 cells
of the declared placement pivot. Large worlds use multiple sparse volumes;
their theoretical cell count is not a residency promise. Bounds, not an up
axis, define the volume.

```rust
pub struct CellWire {
    pub material_id: u16,
    pub density_q8_8: i16,
}

pub struct LocalCellPoint(pub [i32; 3]);
pub struct LocalCellAabb {
    pub min: LocalCellPoint,
    pub max: LocalCellPoint,
}
pub struct BrickCoord(pub [i32; 3]);
pub struct BrickAabb {
    pub min: BrickCoord,
    pub max: BrickCoord,
}
```

Both AABBs are half-open and require `min < max` on every axis.

### TECH-007 — Parameterized canonical placement representation

Implements: REQ-003, REQ-019, REQ-021, REQ-028, REQ-036, REQ-043

Each world freezes one `PlacementFixedFormat` before registry validation:

```rust
pub struct SimulationUnitId([u8; 16]);

pub struct PlacementFixedFormat {
    fractional_bits: u8,
    cell_extent_raw: u32,
    simulation_unit: SimulationUnitId,
}
```

`fractional_bits` is in `0..=16`. A placement scalar is a signed `i32` whose
mathematical value is `raw / 2^fractional_bits` consumer-defined simulation
units. `cell_extent_raw` is in `1..=i32::MAX` and states that one local cell
edge spans exactly that many raw placement increments. `SimulationUnitId` is
a consumer-defined semantic identity; every 16-byte value is valid and Moria
does not interpret it as meters, seconds, mass, force, or any other physical
quantity. The three fields, their canonical bytes, and the canonical-math
contract digest are included in the per-world configuration fingerprint,
genesis root, checkpoint, replay header, and divergence artifact. Restore or
replay with any mismatch fails `ContractMismatch`.

The placement range in simulation units is therefore
`i32::MIN / 2^fractional_bits ..= i32::MAX / 2^fractional_bits`; it is not
hard-coded to a 1/256-cell scale. Genesis validates with checked `i64`
arithmetic that every volume corner relative to its pivot, multiplied by
`cell_extent_raw`, and every translated/rotated corner remain representable as
`i32`. A later placement or collision input is rejected on the same check.
Changing format is a new genesis, never an in-place conversion.

```rust
pub struct PlacementScalar(i32);
pub struct WorldPointQ(pub [PlacementScalar; 3]);
pub struct WorldVectorQ(pub [PlacementScalar; 3]);
pub struct WorldAabbQ {
    pub min: WorldPointQ,
    pub max: WorldPointQ,
}
pub struct SegmentQ {
    pub start: WorldPointQ,
    pub end: WorldPointQ,
}
pub struct TurnQ32(u32);

impl PlacementScalar {
    pub fn from_raw(raw: i32) -> Self;
    pub fn get(self) -> i32;
}
impl TurnQ32 {
    pub fn from_raw(raw: u32) -> Self;
    pub fn get(self) -> u32;
}
impl SimulationUnitId {
    pub fn from_bytes(bytes: [u8; 16]) -> Self;
    pub fn as_bytes(&self) -> &[u8; 16];
    pub fn to_bytes(self) -> [u8; 16];
}
impl PlacementFixedFormat {
    pub fn try_new(
        fractional_bits: u8,
        cell_extent_raw: u32,
        simulation_unit: SimulationUnitId,
    ) -> Result<Self, NewtypeValueError>;
    pub fn fractional_bits(self) -> u8;
    pub fn cell_extent_raw(self) -> u32;
    pub fn simulation_unit(self) -> SimulationUnitId;
}
```

`PlacementScalar::from_raw`/`get`, `TurnQ32::from_raw`/`get`, and
`SimulationUnitId::{from_bytes,as_bytes,to_bytes}` are the only scalar/unit
construction and inspection paths. `PlacementScalar` is not interchangeable
with cell coordinates, density, orientation components, participant values,
or render floats. No `From<f32>`, `Into<f32>`, implicit numeric conversion, or
float-taking canonical constructor exists. Presentation has separately named
one-way conversion functions that require an explicit `PlacementFixedFormat`
and can never return a canonical type.
`PlacementFixedFormat::try_new` rejects a split above 16 and a zero or
`2_147_483_648..=u32::MAX` cell extent; no unchecked aggregate construction
is public.

Local lattice coordinate `c` becomes placement raw value
`checked_i64(c) * cell_extent_raw`, then must fit `i32`. Placement is exactly
`world = translation + rotate(orientation, local_raw - pivot_raw)` and its
inverse is
`local_raw = pivot_raw + rotate_transpose(orientation, world - translation)`.
Every subtraction, product, reduction, and addition uses TECH-071; no
operation may reassociate, reduce early, saturate, or use floating point to
avoid a typed canonical failure. Exact rational comparison in collision
interval clipping and SAT depth comparison uses checked signed 128-bit
two's-complement semantics, implemented in WGSL by reviewed four-word `u32`
helpers.

Orientation remains a canonical quantized unit quaternion
`QuatQ14Wire([i16; 4])`, component order `(x,y,z,w)`, scale
`S = 16,384`. Registration and composition use TECH-071 multiplication,
square root, division, and rounding in this exact sequence:

1. registration treats the four input components as one integer vector;
   composition computes the raw Q2.28 Hamilton product in this exact order:
   `x=aw*bx+ax*bw+ay*bz-az*by`,
   `y=aw*by-ax*bz+ay*bw+az*bx`,
   `z=aw*bz+ax*by-ay*bx+az*bw`,
   `w=aw*bw-ax*bx-ay*by-az*bz`, evaluating terms left-to-right;
2. calculate the exact positive integer squared norm
   `N = x*x+y*y+z*z+w*w` with checked signed 64-bit products/sums; reject
   `N == 0`;
3. for each component magnitude `a`, compute the exact
   `round_ties_even(a*S/sqrt(N))` without first truncating `sqrt(N)`: a
   15-step binary search finds the largest `q in 0..=S` for which
   `q*q*N <= a*a*S*S`; compare `4*a*a*S*S` with
   `(2*q+1)*(2*q+1)*N` in checked `i128`, choosing `q`, `q+1`, or the even
   one on equality (`q == S` cannot increment), then restore the component
   sign;
4. require every result component to fit `i16` and require the quantized-unit
   shell
   `abs(rx*rx+ry*ry+rz*rz+rw*rw - S*S) <= 32,769`; failure of this
   postcondition is `CanonicalFailure::InvalidOrientation`;
5. choose the sign whose first nonzero component in `(w,x,y,z)` is positive;
6. apply the same procedure after every composition.

This comparison is the normative square-root rounding algorithm; an
implementation may not substitute division by `isqrt(N)`. In particular,
registration input `(1,1,0,0)` normalizes to
`(11585,11585,0,0)` rather than remaining length `sqrt(2)`. The shell bound
follows from rounding four exact normalized components by at most one half each.
Inverse negates `(x,y,z)` and repeats sign canonicalization; it does not
renormalize because the squared norm is unchanged. The algorithm is closed
over `QuatQ14Wire` and cannot accumulate backend-dependent drift.

Canonical vector rotation does not assume that the quantized components have
an exactly representable Euclidean length. From the stored components it
recomputes `D = x*x+y*y+z*z+w*w` and builds this signed rational rotation
numerator:

```text
[ D-2(yy+zz)   2(xy-wz)    2(xz+wy)   ]
[ 2(xy+wz)     D-2(xx+zz)  2(yz-wx)   ]
[ 2(xz-wy)     2(yz+wx)    D-2(xx+yy) ]
```

The denominator of every entry is the same exact positive `D`. This is the
scale-independent quaternion rotation formula, so before the final
fixed-point rounding the rational transform is orthogonal even when the
stored quaternion lies anywhere in the permitted quantized-unit shell. Each
numerator term is calculated exactly in `i64`. For each output component, the
three numerator×raw-vector products are checked and summed left-to-right in
the displayed order, then divided once by `D` with round-to-nearest,
ties-to-even.
Inverse rotation uses the transpose of this same numerator and denominator; it
does not rebuild a second matrix from a rounded inverse. Placement is exactly
the raw-unit sequence defined above. Collision, CPU oracle, WGSL, persistence
verification, and replay all use this sequence.

The declared 4,095-cell maximum radius makes the worst representable
one-component orientation quantization step, including final placement-raw
rounding at the selected `cell_extent_raw`, less than one cell. Retained
generated proofs cover the unit-shell
postcondition, rational orthogonality, transpose inverse, composition closure,
and this displacement bound. Float transforms are one-way derived
presentation values and are never accepted back as canonical placement.

```rust
pub struct QuatQ14([i16; 4]); // private x, y, z, w
pub struct PlacementQ {
    pub translation: WorldPointQ,
    pub orientation: QuatQ14,
}

impl QuatQ14 {
    pub fn try_from_components(
        components_xyzw: [i16; 4],
    ) -> Result<Self, CanonicalFailure>;
    pub fn components(self) -> [i16; 4];
    pub fn try_compose(self, rhs: Self) -> Result<Self, CanonicalFailure>;
    pub fn inverse(self) -> Self;
    pub fn try_from_axis_turn(
        axis: WorldVectorQ,
        angle: TurnQ32,
        format: PlacementFixedFormat,
    ) -> Result<Self, CanonicalFailure>;
}
```

`QuatQ14` has no public field or unchecked constructor; its methods apply the
normalization, composition, sign, shell, and inverse rules above.
`try_from_axis_turn` first validates and normalizes the distinct placement
vector with TECH-071's exact axis procedure below. A zero vector returns
`CanonicalFailure::ZeroAxis`; any failure of a checked axis intermediate or
its Q1.30 result returns `CanonicalFailure::UnrepresentableAxis`, never a
partially constructed orientation. The method performs this axis validation
even when the requested angle is zero.

The axis-angle half-angle is the unsigned integer
`TurnQ32::from_raw(angle.get() >> 1)`: this is floor division of the canonical
word by two, discards the low bit, and does not sign-extend or choose another
representative modulo one turn. Consequently an exact full turn has the same
wire word as zero and constructs the same identity orientation; the maximum
word `0xffff_ffff` uses half-angle `0x7fff_ffff`. The method evaluates that
half-angle through TECH-071's canonical sine/cosine, multiplies each normalized
Q1.30 axis component by the Q1.30 sine with one checked `i64` product and one
ties-to-even reduction by `2^30`, uses the Q1.30 cosine as `w`, and applies the
same exact four-component registration normalization above directly to those
signed intermediates. It does not narrow them to `i16` first. This is the only
trigonometric path into canonical orientation; consumers may instead provide
already quantized components through `try_from_components`.
`WorldAabbQ` is half-open. Facade admission validates its public coordinate
fields and every `PlacementQ` use, so constructing an aggregate record cannot
bypass domain, range, or AABB rules.

### TECH-071 — Canonical fixed-point math library

Implements: REQ-007, REQ-028, REQ-036, REQ-043

`src/canonical/math/` and `assets/shaders/canonical/math/` are one named
component, `moria-fixed-v1`. The CPU implementation is generic over
`FixedI32<const FRACTIONAL_BITS: u8>`; the WGSL source uses the same generated
operation bodies with a validated `FRACTIONAL_BITS` pipeline override. The
runtime facade dispatches the frozen world's `0..=16` split to that generic
implementation. The library is the only implementation allowed for canonical
placement arithmetic, quaternion construction/composition, collision
reductions, and canonical participant placement effects.

This integer contract keeps golden math, replay, and hash fixtures portable
across CI hosts and agent worktree runners and prevents a GPU driver update
from redefining their expected bytes. That fixture portability is narrower
than a whole-simulation cross-machine determinism claim; the latter is not in
scope.

For raw signed `i32` operands and split `F`:

- add/subtract negate and absolute value use checked exact integer operations;
- multiply computes one checked signed `i64` product, divides by `2^F`, and
  rounds to nearest with ties to the even retained raw integer;
- divide computes the checked signed `i64` numerator `a * 2^F`, rejects zero
  denominator, and rounds the exact quotient to nearest with ties to even;
- square root rejects negative input, forms the exact nonnegative `u64`
  radicand `raw << F`, selects `floor_sqrt` by a fixed 32-step high-to-low bit
  search, and chooses the nearer adjacent integer by exact squared-distance
  comparison, selecting the even result on equality;
- narrowing, signed right shift, and rational reduction use the same
  round-to-nearest/ties-to-even primitive; floor division is a separately
  named operation and is used only where TECH-007/051 explicitly require it.

Every output must fit its declared wire width. Overflow, division by zero,
invalid shift/split, negative square root, and nonrepresentable output map to
stable `CanonicalFailure` tags. Saturation is forbidden except for an input
verb that explicitly names saturation. WGSL realizes signed `i64` multiply,
add, compare, shift, and division with reviewed two-word `u32` helpers; native
shader `i64` is not a baseline dependency.

Canonical trigonometry accepts `TurnQ32`, where `0` is zero turns and `u32`
wrap is one full turn, and returns `(sin, cos)` as two signed Q1.30 `i32`
words. The following integer recurrence is normative; generated Rust/WGSL may
implement it but may not define or alter it.

Let `a = angle.get()` widened to `u64`, and let one full turn in the internal
signed angle domain be `2^62`. Reduction chooses the nearest quadrant center:

```text
q_unwrapped = (a + 0x2000_0000) >> 30       // 0..=4
q           = q_unwrapped & 3               // 0,1,2,3
r32         = i64(a) - i64(q_unwrapped << 30)
z0          = r32 << 30                     // 2^62 units per turn
```

Thus `r32` is in `[-2^29, 2^29-1]`. An exact midpoint between two quadrant
centers belongs to the center with the greater unwrapped index and therefore
has residual `-2^29`; this also specifies the wrap midpoint near one turn.
Exact quadrant-center inputs have zero residual.

The signed Q2.61 gain-inverse and arctangent-in-turns table contain entries
`round_ties_even(K^-1 * 2^61)`, where
`K = product(i=0..31, sqrt(1 + 2^(-2i)))`, and
`round_ties_even(atan(2^-i)/(2*pi) * 2^62)` for `i=0..31`. Those integers are
generated once by the repository's arbitrary-precision fixture generator,
checked in as canonical source data, hashed into the arithmetic contract, and
emitted into both Rust and WGSL; runtime libm, shader transcendental
instructions, fused operations, or driver-provided approximations are
forbidden.

Initialize `x0 = gain_inverse_q61`, `y0 = 0`, and `z0` as above. For
`i = 0..31` in increasing order, first snapshot `(xi, yi, zi)`, define
`sx = floor_div(xi, 2^i)` and `sy = floor_div(yi, 2^i)`, and perform exactly
one of these simultaneous checked-`i64` updates:

```text
if zi >= 0:                         // zero residual takes this branch
    x(i+1) = xi - sy
    y(i+1) = yi + sx
    z(i+1) = zi - atan_table[i]
else:
    x(i+1) = xi + sy
    y(i+1) = yi - sx
    z(i+1) = zi + atan_table[i]
```

`floor_div(v, 2^i)` rounds toward negative infinity; a language or shader
right-shift is usable only when proved to have exactly that result. Updates may
not observe another field's new value from the same iteration. After iteration
31, reduce `x32` and `y32` from Q2.61 to Q1.30 by one division by `2^31`,
rounding nearest with ties to even. Call those retained words `(c, s)` and
apply the exact checked quadrant remap:

```text
q = 0: (sin, cos) = ( s,  c)
q = 1: (sin, cos) = ( c, -s)
q = 2: (sin, cos) = (-s, -c)
q = 3: (sin, cos) = (-c,  s)
```

No clamp, second normalization, early reduction, reassociation, or
table-dependent branch is permitted.

Axis normalization treats the three signed `i32` raw components as one exact
integer vector, independent of the placement fractional split because a
common scale cancels. It forms
`N = abs(x)^2 + abs(y)^2 + abs(z)^2` with checked `u64` products and sums. For
each magnitude `a`, it uses a 31-step binary search for the largest
`q in 0..=2^30` satisfying `q*q*N <= a*a*2^60`, compares the adjacent
candidates by comparing `4*a*a*2^60` with `(2*q+1)*(2*q+1)*N`, chooses `q`
when the left side is smaller, `q+1` when it is greater, and the even one on
equality (`q == 2^30` cannot increment), then restores the sign. All
comparison products use checked `u128` (reviewed four-word `u32` helpers in
WGSL).
`N == 0` is `ZeroAxis`; any overflow, impossible comparison, or result outside
signed Q1.30 is `UnrepresentableAxis`. Retained exhaustive range proofs show
that every nonzero three-`i32` public axis is representable in v1; the latter
tag remains the fail-closed outcome for decoder corruption or implementation
contract violation.

The public boundary exposes only the distinct raw canonical types from
TECH-007. The internal generic type cannot be constructed with an unchecked
split, and no canonical module depends on `f32`/`f64`. A Clippy deny rule and a
source audit fail if canonical or canonical-WGSL modules contain floating
types, float literals, implicit scalar casts, or calls outside
`moria-fixed-v1`.

## Canonical bytes and commitments

### TECH-008 — Canonical wire encoding

Implements: REQ-017, REQ-028, REQ-032, REQ-034, REQ-038

Canonical encoding version `moria-canonical-v1` is a hand-written,
schema-tested binary format:

- unsigned and signed integers are fixed-width little-endian two's complement;
- booleans and enums are `u8` tags with rejected unknown values;
- digests are exactly 32 bytes;
- sequences use a `u32` element count and elements in their specified order;
- optional fields use a `u8` presence tag followed by the value;
- no platform-sized integer, float, string, map, implicit padding, or
  serde-derived layout is allowed in canonical bytes;
- every decoder rejects trailing bytes, nonminimal variants, excessive
  lengths, invalid tags, and arithmetic overflow.

Human labels are bounded UTF-8; correlation metadata is a bounded ID and opaque
byte payload. Both are noncanonical and do not appear in replay identity or
hashes. Contract, schema, arithmetic, shader, and hash-domain versions are
fixed digests in genesis. CPU encoding and WGSL wire layout have byte-for-byte
fixtures for every record.

### TECH-009 — Merkle commitment

Implements: REQ-001, REQ-017, REQ-028, REQ-032, REQ-034, REQ-038

BLAKE3-256 is the canonical hash algorithm, implemented with 32-bit operations
on CPU and WGSL. Every node hashes:

```text
"moria/v1/<domain>" || canonical_length || canonical_payload
```

Distinct domains cover genesis, material registry, base source, brick,
scar-leaf, radix-node, volume metadata, simulation domain, allocator state,
participant commitment, outcome list, tick batch, tick state, and world root.
The world root combines child commitments in stable ID/key order. Derived
presentation, lifecycle cache state, physical slots, receipt IDs, timings, and
telemetry are excluded.

The canonical material-registry payload contains material ID and occupancy
class/threshold. Surface style and asset handles are in the separately
versioned derived-presentation registry and are excluded from the world root.
The configuration fingerprint is a `ContractDigest` calculated as
`BLAKE3("moria/v1/configuration" || canonical_length ||
canonical_configuration_bytes)`. Those bytes contain, in displayed/sorted
field order, every `CanonicalContract` digest; the exact TECH-007 placement
format; TECH-071 arithmetic/table digest; canonical resource limits that can
produce a transition outcome; material, volume, content, and input-source
canonical descriptors; and participant contract/input/event/representation/
RNG/rollback/failure descriptors. Presentation policy, asset handles,
telemetry, adapter/driver identity, and diagnostic candidate fault plans are
excluded. The genesis domain includes this fingerprint and rejects any
configuration whose canonical fields were omitted from it.

Hashing is incremental. A changed brick recomputes its leaf and 26 four-bit
radix ancestors; a changed volume recomputes its volume leaf and the world
registry path. Unchanged node hashes are retained. A tick reports changed leaf
and node counts, and a test fails if a one-brick fixture schedules unrelated
volume hashes.

## Sparse authoritative state

### TECH-010 — Logical sparse representation

Implements: REQ-001, REQ-003, REQ-004, REQ-014, REQ-018, REQ-029

Each volume is the immutable tuple:

```text
VolumeState {
  id, kind, domain, base_authority, placement, revision,
  scar_root, simulation_regions, retired
}
```

`kind` is `Static` or `Dynamic`; static placement cannot change after genesis.
The base authority describes homogeneous content or a content-addressed brick
manifest. The scar root is a persistent 4-bit radix tree keyed by:

```text
VolumeId:u64 || zigzag(bx):u32 || zigzag(by):u32 || zigzag(bz):u32
```

The 104-bit key has exactly 26 radix levels. A leaf is either a uniform-cell
override or a complete canonical brick. Absence means “obtain this brick from
the exact base authority,” never “empty.” A scar leaf is omitted only when its
payload is byte-identical to verified base content.

The world root includes sorted immutable registries, volume roots and
placements, canonical simulation-domain union, `next_volume_serial`,
participant commitments (including their ordered canonical RNG-state
commitments), the canonical frontier position (`Genesis` or
`Confirmed(Tick)`), placement format, configuration fingerprint, and contract
identities. `Genesis` is a real pre-tick
position and is not encoded as tick zero. Runtime residency and readiness are
a separate cache indexed by
`(base digest, volume, brick)`.

### TECH-011 — Deterministic tick ordering and conflict rules

Implements: REQ-011, REQ-017, REQ-027, REQ-031, REQ-033, REQ-036, REQ-043

Tick eligibility is defined over the closed `FrontierPosition` in TECH-070:
`Genesis` admits exactly `Tick::from_raw(0)`, and `Confirmed(t)` admits exactly
`t.get().checked_add(1).map(Tick::from_raw)`. Overflow leaves no eligible next
tick. Genesis publication
therefore does not consume tick zero. A sealed `TickBatch` contains bounded
inputs whose unique key is `(phase:u8, source_id:u32, source_sequence:u32)`.
The fixed phase order is:

0. opaque participant input delivery;
1. volume create/retire and simulation-domain activation/deactivation;
2. placement changes;
3. direct matter commands;
4. participant-proposed ordinary placement or matter commands.

Inputs are sorted lexicographically by this key. Duplicate keys reject the
whole batch before admission. Source sequence is explicit consumer data; queue
arrival never supplies it. Direct inputs use their registered high-bit-clear
`InputSourceId`. Participant effects use
`0x8000_0000 | ParticipantId` as the order source in a disjoint namespace and
a bounded local sequence. Participant commitments are derived products, not
batch inputs; they combine separately in `ParticipantId` order.

For attempted tick `n`, define `SourceState(n)` as the canonical genesis state
when `n == 0`, otherwise confirmed `State[n - 1]`. Participant preparation
reads `SourceState(n)` plus tick `n`'s phase-zero input. It does not observe
same-tick lifecycle, placement, or direct-matter effects; all such effects,
including its own proposals, compose into `State[n]`. This read-before-write
rule is part of the transition version. It also cannot
observe another participant's same-tick state, effects, or opaque events.
Registration rejects a dependency declaration or adapter requiring a
same-tick predecessor. V1 has no participant DAG, handoff pass, prior-feedback
buffer, or conflict callback.

All revision and source-hash preconditions for tick `n` are evaluated against
`SourceState(n)`.
Eligible successful commands then compose in canonical order on a staged
state; for overlapping writes the later canonical command sees the earlier
staged cell. A failed command contributes a canonical outcome at its order
position but no writes or revision advance. Tick-global inability to bind
content, participants, arithmetic contract, or canonical resources yields the
closed `FailedNoAdvance` receipt error; there is no partial tick publication.

This is also the complete participant-effect conflict policy. Participant
effects occupy phase 4 in `(ParticipantId, local_sequence)` order. Overlap is
legal and composes by the rule above; stale or otherwise unmet preconditions
fail only that effect. V1 adds no conflict graph, ownership lock, handoff
buffer, arbitration callback, or automatic retry.

### TECH-012 — Atomic mutation and revision rules

Implements: REQ-011, REQ-017, REQ-025, REQ-033

The canonical matter commands are `Erase`, `Place`, and `Patch`. Each targets
one live volume and at most 64 bricks / 32,768 cells.

- `Erase` applies a bounded local AABB, sphere, or stamp mask and either sets
  selected cells to canonical empty or subtracts an explicit Q8.8 density
  amount with specified saturation at empty.
- `Place` applies the same shapes and replaces selected cells with one valid
  `CellWire`.
- `Patch` supplies sorted unique `(local_cell, CellWire)` pairs.

Shapes are discretized by the world's TECH-007 placement format and TECH-071
integer rules. Empty target sets are valid no-op outcomes and do not advance a
revision. All nonempty targeted cells,
base bricks, destination slots, and output sizes are resolved before any new
root is constructed. A command-level validation or capacity failure marks the
command failed and writes nothing. A successful nonempty command advances its
volume revision exactly once, even across many bricks. Create, retire, and a
dynamic placement change likewise advance only their named volume lifecycle
revision. `u64` revision exhaustion is a typed failure.

### TECH-013 — Multi-phase construction and atomic publication

Implements: REQ-001, REQ-005, REQ-011, REQ-033, REQ-035

A tick attempt has these ordered phases:

1. decode and structural validation on CPU before ownership transfers;
2. pin `SourceState(n)`, exact base chunks, participant resources, and pool
   permit;
3. run participant preparation against the pinned source commitment;
4. stable-sort and validate direct and proposed effects;
5. mark touched bricks and compute exact capacities with checked prefix scans;
6. materialize complete old brick values into unreferenced work slots;
7. apply each brick's commands in canonical order;
8. hash changed leaves and copy-on-write radix ancestors;
9. validate all diagnostic, overflow, participant, and output records;
10. encode outcomes and calculate the new world root;
11. await GPU completion and bounded outcome/hash mapping;
12. deliver one generation-tagged candidate envelope through the reserved
    render-to-main completion bridge, then atomically swap the main-world
    `FrontierBundle` containing the GPU root token and participant state
    tokens, confirm the receipt, install the semantic replay-log record, and
    emit observations. The ordinary live-stream append then follows TECH-047;
    a correction instead satisfies TECH-048's durable branch precondition
    before this swap.

Dependent GPU phases are ordered dispatches on one queue. No shader uses a
cross-workgroup spin protocol. Before step 12, only private slots refer to the
candidate root. Any failure, missing bridge reservation, or device-generation
mismatch discards/retire-queues those slots and leaves `SourceState(n)`, its
revisions, participant state, snapshots, and hash live. Step 12 occurs in the
exclusive main-world publication system specified by TECH-032. The root token
names already completed device objects retained in the render world, so
subsequent extracted work can use it without a second render-world “live”
swap. Readers acquire either the old or new immutable `FrontierBundle`, never
a mixture.

## Snapshot sharing and rollback

### TECH-014 — Persistent roots and bounded rollback window

Implements: REQ-018, REQ-029, REQ-035, REQ-037, REQ-043

`RollbackConfig.capacity_ticks` defaults to 32, must be at least 20, and is
bounded above by the configured canonical-memory budget. Each confirmed
frontier retains:

```text
tick, world_root_hash, GPU root handle, immutable metadata root,
tick-batch digest, outcome digest, participant commitments,
opaque participant state tokens and snapshot metadata
```

The separately retained genesis root has `FrontierPosition::Genesis`, is the
source for tick zero and replay bootstrap, and does not count as one of the
required 20 confirmed-tick frontiers. Every record in the rollback deque has
`FrontierPosition::Confirmed(tick)` and the displayed `tick` is that value.
The frontier is O(changed bricks × radix depth) additional logical state.
Installing one is an O(number of registries + participants) root-handle swap;
it does not enumerate material bricks. Active live, retained, replay, query,
checkpoint, and participant leases pin roots. A root leaves the window only
after capacity eviction and no pin remains.

If 20 frontiers cannot be retained under the declared canonical-state budget,
genesis fails. Once running, deterministic logical-budget preflight prevents a
tick from confirming if it would violate the minimum retained window. It never
evicts a reachable frontier based on completion timing.

### TECH-015 — Stable compaction and physical reclamation

Implements: REQ-004, REQ-018, REQ-029, REQ-033, REQ-037

Candidate keys and keep predicates use fixed slots followed by portable
`mark -> hierarchical exclusive scan -> scatter`. Tile width is 128
invocations with two elements per lane. Tile totals are recursively scanned in
separate ordered dispatches. Inactive lanes participate in every barrier with
the identity value.

Every result reports `total`, `written`, and `overflowed`; overflow aborts the
candidate rather than truncating it. Sorting is stable LSD radix sort over
fixed-width keys, four bits per pass. No atomic append order enters canonical
state.

Physical nodes and bricks have `(slot, generation)` handles. Reclamation first
removes all new references, then waits for root pins and the queue completion
of every prior reader, then increments the generation and returns the slot.
Generation wrap permanently retires the slot. A stale handle is rejected
before encoding. Physical free-list order is deliberately noncanonical; all
canonical resource outcomes are decided against logical configured budgets
before physical allocation.

### TECH-016 — Coordinated participant frontier

Implements: REQ-006, REQ-029, REQ-030, REQ-033, REQ-035, REQ-037, REQ-043

Every participant registers exactly one strategy:
`PerTickSnapshot { max_bytes }` or
`ReconstructibleFromCanonicalStateAndLog { max_replay_ticks }`.
Its canonical state is participant-owned in meaning and representation, but it
must be encapsulated in an immutable opaque state token whose lifetime Moria
can pin. There is no adapter-global mutable canonical state. A token is bound
to `(participant, contract, frontier: FrontierPosition, world_root,
commitment, device_generation?)`, and only the originating adapter may inspect
it. Genesis tokens are bound to `FrontierPosition::Genesis`; the token
produced by attempted tick `n` is bound to
`FrontierPosition::Confirmed(n)`. No sentinel tick represents genesis.

For every attempted tick the adapter receives a lease to the source token,
source root hash, bounded input slice, and canonical artifact leases. It
constructs a new uninstalled token and returns a bounded ordered effect list,
a bounded ordered opaque event list, a 32-byte participant commitment, and,
for each declared RNG stream, the canonical RNG-state commitment specified
below. Effects are ordinary commands and have no privileged mutation path.
Events are participant-owned output carried in canonical participant records
and the confirmed tick receipt; they never enter Moria's observation stream or
feed another participant in the same tick.
Preparing a token may not mutate the source token. A tick confirms only when
the exclusive coordinator installs one immutable `FrontierBundle` containing
the candidate root and every prepared participant token. Before that swap all
tokens are private; after it the old bundle remains pinned for readers and
rollback. Thus participant installation cannot partially commit independently
of substrate publication.

A rollback or correction creates a private `CorrectionContext` containing
tokens restored from the target frontier. Each replayed tick produces the next
private token in that context. Success installs only the final bundle; failure
or cancellation drops every private token after its CPU/GPU leases drain and
leaves the original live bundle untouched. Snapshot restore and reconstruct
operations therefore return staged tokens; they never mutate an in-place
participant. Device-generation loss terminally invalidates staged GPU tokens
from that generation. Old-generation completion may release resources but
cannot enter a live or correction bundle.

Every CPU and GPU participant operation uses the same bounded lifecycle:

```text
Reserved(source pin + destination bytes)
  -> Preparing | RestoringSnapshot | Reconstructing
  -> PreparedPrivate
  -> InstalledInFrontier
  \-> Failed -> Aborting -> DrainingLastUse -> Reclaimed
```

Only `PreparedPrivate` may enter a bundle, and installation is the host pointer
swap rather than an adapter callback. A sink completion moves to
`PreparedPrivate`; duplicate completion is rejected. Cancellation is accepted
only before preparation is submitted. After submission it suppresses
installation, drains the token, and returns its fixed permit. Descriptor maxima
for source state, destination state, effects, opaque events, snapshot bytes,
replay ticks, and artifact leases are reserved before the operation; no
callback may grow them. Effects and events use separate Moria-owned fixed-slot
sinks with exact aggregate byte counters. A completion cannot return a
consumer-owned `Vec`, map, diagnostic string, or allocation; it fills those
sinks and one bounded diagnostic record.

For `PerTickSnapshot`, each prepared token also exposes immutable
`SnapshotMetadata { uncompressed_bytes, digest }` and a bounded asynchronous
export operation. Moria pins the token for the rollback window. Participant
code owns the snapshot schema, while Moria owns the lifecycle of the handle and
copies verified snapshot bytes into `CheckpointStore` for durable checkpoints
as specified by TECH-045. The participant is not allowed to substitute a
durable external locator. `ReconstructibleFromCanonicalStateAndLog` tokens
declare their maximum replay prefix and must reproduce every intermediate
commitment from canonical genesis/frontier plus exact log bytes. A durable
checkpoint owns content-addressed copies of the required replay records as
specified by TECH-044 through TECH-049; an in-memory range or digest without
those bytes is not a reconstruction source.

Moria itself has no RNG algorithm or RNG state. A participant using randomness
that can affect canonical output must list every stream in its genesis
descriptor:

```text
RngContract {
  stream_id: nonzero u32,
  algorithm_id: 16 bytes,
  algorithm_version: u32,
  algorithm_contract_digest: 32 bytes,
  state_schema_digest: 32 bytes,
  seed_bytes: 0..=64 canonical bytes
}
```

The referenced algorithm contract must completely specify seed decoding,
state bytes, next-state/output transition, rejection sampling, and exhaustion.
Each participant commitment contains, in `stream_id` order,
`(stream_id, state_byte_len:u32, BLAKE3(state_bytes))`. Snapshot bytes contain
the complete state bytes for every stream. A reconstructible participant must
derive them from the descriptor seed and canonical log and reproduce those
digests. OS entropy, wall clock, thread identity, and undeclared streams are
conformance failures. These descriptors and state commitments are included in
genesis, world hashing, retained frontiers, checkpoints, replay, restore, and
determinism evidence; a 32-byte participant commitment alone is not treated
as an RNG specification.

Moria defines no canonical representation for participant-owned velocity,
acceleration, mass, force, energy, time, or any other physical quantity.
Every deterministic participant lists its non-placement representations in
its genesis descriptor as sorted unique
`ParticipantRepresentationContract { representation_id, quantity_schema,
representation_contract }` records. The referenced contract completely
specifies width, unit identity, scale/fractional split where applicable,
rounding, overflow, canonical byte encoding, and CPU/GPU implementation. The
records are part of the participant descriptor commitment, genesis
configuration fingerprint, checkpoint, and replay header. Opaque participant
input/state/event bytes are accepted only under those bound schemas and
contracts. Participant effects that set Moria placement must use the world's
TECH-007 representation; a participant-specific physical representation never
implicitly converts into it.

Missing, oversized, wrong-source, duplicate, late-generation, or divergent
products cause `FailedNoAdvance` or rollback failure. Participant completion
order is irrelevant: products occupy preassigned `ParticipantId` slots and are
combined in ID order. Moria never interprets participant behavior or RNG
meaning, but it validates every declared bound, identity, digest, and lifecycle
transition. The descriptor's closed `ParticipantFailurePolicy` controls
whether such a failed canonical operation leaves the world retryable at its
last frontier or terminally fails it; TECH-029 defines the complete matrix.
No policy can omit the participant, reuse its prior token as the next tick's
state, synthesize an empty commitment, or publish a partial bundle.
