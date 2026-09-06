# Moria product design

## 1. Product definition

Moria is reusable voxel-world infrastructure for Rust and Bevy consumers. A
game or tool installs Moria, supplies material content, and interacts with that
content through one public consumer contract. Moria owns material truth,
bounded access to it, admitted changes, lifecycle, persistence seams,
collision/occupancy truth, and presentation derived from that truth.

The product is a substrate, not a game. It does not supply a player, camera,
world-generation recipe, physics engine, damage model, progression system, or
game-specific content. Validation executables may accompany the substrate, but
they are ordinary consumers of the same contract and do not define the
product's identity or completion.

Moria's central promise is:

> A consumer can host sparse, continuous three-dimensional material volumes;
> inspect, move, and edit them without privileged storage access; and rely on
> queries, collision, persistence, and presentation to refer to the same
> authoritative matter. Given the same canonical genesis and ordered,
> tick-stamped inputs, every qualified backend produces the same canonical
> simulation state and hash sequence; a consumer can roll back and replay a
> recent confirmed interval without copying or traversing the whole world.

Authoritative matter is designed to remain GPU-resident at scale. Consumer
interaction therefore uses an asynchronous-capable command, query, observation,
and telemetry boundary rather than direct storage or buffer ownership.
Residency is a performance direction: it must let Moria's owned work, and
downstream behavior engines where feasible, operate without making a full CPU
mirror or synchronous readback the normal route to world truth. It does not
make those behavior engines or their policies part of Moria.

Deterministic authority is mandatory, not a selectable operating mode. It is a
contract over canonical state, inputs, transitions, rollback, hashes, and
qualified results; it does not require deterministic rendered pixels, a
networking implementation, or a built-in behavior engine.

## 2. Design principles

### 2.1 Matter is authority

**Requirement: REQ-001. Authority: C-003, C-016, D-006, D-007.**

A material volume is the source of truth. Surface meshes, dressing, collision
debug shapes, slice views, and other visualizations are disposable projections
of a known canonical tick and matter revision. They may be rebuilt, delayed,
discarded, or differ between backends without changing what exists.

Collision and occupancy never infer truth from presentation. Persistence never
saves derived presentation as truth. A visual result that cannot survive an
edit and honest rebuild does not satisfy the product.

Every byte included in the canonical simulation hash, and every value capable
of changing such a byte in a later tick, is canonical state. Everything else
is a derived cache or non-authoritative observation. A derived artifact may
read canonical state but may never feed readiness, contents, or completion
order back into simulation.

### 2.2 One contract for every consumer

**Requirement: REQ-002. Authority: C-003, C-013.**

An external game, editor, benchmark, and in-repository demo all:

- configure the same public facade;
- inject base material content through the same content seam;
- request bounded readiness and inspection;
- submit the same admitted commands;
- observe the same lifecycle and change notifications; and
- receive the same errors and telemetry.

No validation mode may bypass admission, inspect internal storage, mutate a
private CPU mirror, or treat a generated mesh as the world.

### 2.3 Volume, not terrain

**Requirement: REQ-003. Authority: C-002, C-006, C-007, AD-004, AD-005.**

The basic addressable object is a material volume, not a heightmap or a
gravity-aligned landscape. A volume has stable identity, its own local
three-dimensional address space, an extent or addressable domain, and a
placement in a containing world. Static landscape is one use of this model.
Movable material bodies and constructed interiors use the same model.

Nothing in the consumer contract assigns a privileged "up" axis, surface
height, sea level, ground plane, or planetary meaning. Consumers may add those
interpretations.

### 2.4 Sparse cost follows interest and change

**Requirement: REQ-004. Authority: C-004.**

Empty space and untouched homogeneous matter remain cheap. Detailed cost is
paid for materially interesting boundaries, voids, registered assemblies, and
scars, and only while consumer interest or unfinished work requires them.

The product makes this behavior observable through lifecycle and telemetry. It
does not promise an unbounded world with hidden unbounded residency.

### 2.5 Asynchrony is explicit

**Requirement: REQ-005. Authority: C-010, AD-003, D-002, D-007.**

Submitting work is distinct from completing work. A consumer never has to
assume that a GPU operation completed synchronously or that a readback is
immediately available. Commands, queries, derived presentation, and
persistence checkpoints expose their progress, completion, revision, and
failure rather than hiding delay behind direct storage access.

When an external behavior engine already operates on the GPU, this boundary
must support a bounded, authorized path for observing matter and requesting
effects without making a full CPU mirror or synchronous CPU readback the normal
coordination route. GPU compatibility does not grant direct ownership of
Moria's storage: bounds, admission, revisions, completion, and failure remain
the same public contract regardless of where the participating work runs.

Canonical computation may finish asynchronously, but finish order and wall
time never select canonical state. Results become authoritative only through
the transition and publication rules of their numbered tick. A required input
or canonical derivation that is unavailable at that boundary follows a
deterministic fail-closed or no-advance outcome; it is never replaced by empty
matter or a timing-dependent fallback.

### 2.6 Behavior remains external

**Requirement: REQ-006. Authority: C-008, AD-007, D-008.**

External systems may observe matter and lifecycle, perform any computation they
own, and request admitted changes. Moria does not interpret those changes as
gravity, force, fracture, damage, health, growth, fire, fluid flow, or another
behavior vocabulary. The substrate supplies truth and controlled effects, not
the rule that decides an effect should occur.

An external system whose state can affect later canonical results is a
coordinated deterministic participant. It declares how it rolls back,
contributes a canonical commitment, consumes tick-stamped inputs, and publishes
proposed effects through Moria's normal authority boundary. Coordination does
not transfer its state meaning or algorithm to Moria.

### 2.7 Correct first, then optimize continuously

**Requirement: REQ-007. Authority: C-015, D-003, D-004.**

Correct material truth, atomic mutation, honest readiness, and explicit failure
come before performance claims. Optimization may change where or how work runs,
but it may not weaken those public semantics, introduce a second world, or make
unknown matter appear resolved.

Within that constraint, Moria is designed for relentless, measured
optimization. GPU residency exists to make large-scale world work fast, not as
an architectural ornament. Generic extension seams must support the intended
performance shape in which downstream physics and other behavior engines run
primarily on the GPU when feasible, observe Moria truth without privileged
storage ownership, and return requested effects through normal admission. A
consumer may still choose a CPU-oriented engine; behavior placement and policy
remain external.

### 2.8 Determinism is qualified, not presumed

**Requirement: REQ-026. Authority: C-015, D-004.**

Moria distinguishes two guarantees:

- **local replay determinism:** one qualified implementation and backend
  reproduces canonical state, outcomes, and hashes across repeated runs,
  schedules, and rollback/replay; and
- **cross-GPU determinism:** every claimed qualified GPU vendor, driver, and
  backend tuple produces bit-identical canonical results for the retained
  conformance fixture.

Passing locally does not prove cross-GPU conformance. Rendering correctly does
not qualify a backend for deterministic authority. Qualification is explicit,
versioned, tied to the tested hardware/software tuple, and invalidated by a
driver change or relevant canonical-transition change until evidence is
renewed.

## 3. Consumer mental model

Moria exposes the following product concepts. Their precise Rust types and
internal representations belong to technical design.

### World

A world is the consumer-facing container for registered materials, material
volumes, content sources, lifecycle, scars, and observation. It provides the
scope in which identities and revisions are meaningful.

### Material

A material definition gives matter stable identity and the information Moria
needs for its owned responsibilities: whether and how it contributes to
occupancy and surface presentation, plus domain-appropriate presentation
inputs. Consumers may associate their own opaque metadata with a material, but
Moria does not standardize behavioral fields such as hardness, resistance,
wetness, temperature, flammability, health, or damage.

Empty space is distinguishable from occupied material. Material truth also
contains enough shape or coverage information to derive honest boundaries and
edited cut faces without requiring cube rendering as the product look. The
encoding and precision of that shape information are technical choices.

### Material volume

A material volume is a stable, addressable body of material truth. Static and
dynamic volumes have the same inspection, mutation, persistence, and
presentation guarantees.

A dynamic volume additionally has an admitted placement that may change over
time. Moving a volume changes where its matter is encountered; it does not
rewrite the volume's local material content. Movement has no built-in cause or
response. An external behavior system may decide where a volume moves and
request the placement change.

When volumes overlap, inspection preserves the identities of all encountered
volumes rather than silently merging them or choosing game-specific contact
behavior. A consumer or behavior plug-in decides what overlap means.

### Base content source

A base content source supplies matter for bounded parts of a volume when Moria
needs to materialize them. It may read authored content, import content, run a
consumer-owned generator, or combine those approaches. Moria does not know or
prescribe the algorithm.

Every base source declares a stable content lineage that can be compared with
persisted scars. For matter a checkpoint does not carry, the source must be
able to reconstruct the same base content for that lineage, or the consumer
must durably supply that base content alongside the scars. A changed or
non-reconstructable base requires a new lineage and explicit migration or
rebase. This prevents edits made against one base world from being silently
replayed onto a different one while still allowing untouched homogeneous
volume to remain cheap.

### Scar

A scar is a durable change relative to consumer-supplied base matter. It
includes admitted matter edits and the minimum substrate-owned world state
needed to reconstruct changed material volumes and their placements.
Consumer-owned simulation or gameplay state remains the consumer's persistence
responsibility.

### Revision

A revision identifies a committed state of a volume. Successful mutations and
placement changes advance the relevant revision. Query results, observations,
presentation status, and persistence checkpoints identify the revision they
describe, allowing consumers to reason about freshness without seeing storage.

Revisions express causality and freshness; their concrete representation and
cross-volume coordination are technical concerns.

### Interest

Interest is a bounded consumer request to make a portion of one or more volumes
ready for stated uses, such as inspection, collision, or presentation. An
interest source may represent a camera, editor selection, agent, benchmark
route, or background tool. Moria does not assume that interest is camera-shaped
or tied to a single player.

### Receipt and observation

A receipt lets a consumer follow admitted asynchronous work to completion. An
observation reports a committed change or lifecycle transition through a
bounded public channel. Observations are facts about Moria-owned state, not
instructions to an external behavior system.

### Canonical genesis, tick, and input batch

**Requirement: REQ-027. Authority: D-002.**

Verified world construction and base-content installation end by publishing
one versioned canonical genesis state. Tick zero begins after that boundary;
there is no later bootstrap exception.

A tick is the only authority that may advance canonical state after genesis.
Its input is one bounded, versioned `TickBatch`: the complete collection of
canonical inputs admitted for that numbered tick in a deterministic total
order. Submission thread, callback order, worker scheduling, and GPU completion
order cannot change the batch or its meaning. Convenience calls may help a
consumer construct inputs, but they do not publish state outside the tick.

The conceptual promise is `State[t + 1] = Transition(State[t], TickBatch[t])`.
The notation specifies consumer-visible causality, not an implementation
algorithm.

### Canonical state and simulation representation

**Requirement: REQ-028. Authority: D-003, D-006.**

Canonical state contains every value capable of changing a later canonical
result: matter and placements; stable identities; tick, revision, allocation,
and RNG state; simulation-domain membership; content identity; coordinated
participant commitments; and any future-transition input not reproduced from
the canonical log. The canonical hash covers this same influence boundary.

Canonical transition values use integer or precisely specified fixed-point
semantics. This includes simulation-facing coordinates, mutation geometry,
placement translation and orientation, collision facts that drive behavior,
ordering and allocation, and participant values that directly produce
authoritative effects. Arithmetic ranges, overflow, division, rounding,
shifts, normalization, canonical byte encoding, and CPU/GPU/persistence parity
are explicit parts of the qualified contract; backend defaults are not.

Placement supports arbitrary three-dimensional rigid orientation. Its
canonical representation is closed under composition and inverse and cannot
accumulate backend-dependent drift. At the declared maximum supported volume
radius, one orientation quantization step moves a voxel by less than one cell.
Floating-point render or query forms may be derived, but never stored back into
canonical state.

Any RNG that can affect canonical state has an explicit algorithm, seed, and
complete state represented in hashes and rollback. Wall-clock time, thread or
worker identity, OS entropy, I/O completion timing, and presentation readiness
are not implicit transition inputs.

### Confirmed frontier and rollback snapshot

**Requirement: REQ-029. Authority: D-005, D-008.**

The confirmed frontier is the latest canonical tick that Moria and all
coordinated participants have committed and retained according to their
contracts. A rollback snapshot is a persistent reference to canonical state at
such a frontier, not a serialized copy of the whole world.

Moria retains a configurable bounded rollback window with capacity for at
least 20 confirmed ticks. Unchanged world state is shared across retained
frontiers; advancing a tick pays for changed state and the bounded publication
path needed to make it authoritative. Restore installs a retained frontier
without copying or traversing untouched world material, then coordinates every
participant's declared restoration strategy. State reachable from the retained
window or an active replay cannot be reclaimed or reused.

This rollback snapshot is distinct from a durable application checkpoint. The
former provides recent correction and replay; the latter survives application
lifetime boundaries.

### Deterministic participant

**Requirement: REQ-030. Authority: C-008, AD-007, D-008.**

A participant is an external behavior or simulation system whose state or
output can affect canonical results. Registration declares stable identity,
contract and input-schema versions, bounded resource claims, failure behavior,
a canonical hash contribution or reconstruction proof, and exactly one
rollback strategy:

- **per-tick snapshot:** retain bounded participant state at the coordinated
  confirmed frontier; or
- **reconstructible from canonical state and log:** rebuild from the selected
  substrate snapshot and replay, reproducing the original commitments and
  effects.

There is no default strategy. A participant that cannot restore or reproduce
its commitment fails rollback or conformance explicitly. Moria coordinates
frontiers, inputs, commitments, and proposed effects but does not interpret or
repair participant-owned behavior.

### Simulation domain

**Requirement: REQ-031. Authority: D-009.**

The simulation domain is the canonical set of volumes or bounded regions
eligible to affect a tick. It is separate from local render, inspection,
preload, and materialization interest. Activation and deactivation are
tick-stamped canonical inputs and bind the exact content lineage or digest
being activated.

Consumer-defined activity regions may propose membership. Their overlaps
resolve to a deterministic union, so overlap never causes duplicate processing.
A consumer may coordinate a coarse simulation outside a full-physics region;
Moria transports its declared canonical state without defining what "coarse,"
"active," or "physics" means.

### Replay record

**Requirement: REQ-032. Authority: D-006.**

A replay record is a versioned public contract containing canonical genesis
identity and configuration fingerprints, ordered tick batches, participant and
contract identities, sufficient frontier context, and—when recorded for
validation—the expected canonical hash at each tick. Genesis plus the complete
input log reproduces canonical state and its hash sequence.

Replay serves rollback verification, deterministic regression, desync
diagnosis, tests, and captured demonstrations. On divergence, a portable
failure artifact identifies the earliest divergent tick and includes the
genesis identity, contract versions, input prefix, expected and actual hashes,
and backend qualification context. Presentation frames and timing are not part
of replay identity.

## 4. Consumer journey

### 4.1 Configure

**Requirement: REQ-008. Authority: C-001, C-003, D-002, D-003, D-004, D-005, D-008.**

Before a world becomes usable, the consumer:

1. installs the Moria facade in its Bevy application or tool;
2. registers material definitions and presentation inputs;
3. registers one or more static or dynamic material volumes;
4. supplies a base content source for each volume that needs one;
5. registers each canonical participant with its versioned input contract,
   rollback strategy, commitment, resource bounds, and failure behavior;
6. selects persistence, rollback-window, presentation, resource-budget, and
   telemetry policies;
7. declares the canonical arithmetic, content, contract, and backend
   qualification identities required by the session; and
8. verifies and publishes genesis, then observes whether the world becomes
   ready for tick zero or fails.

Missing material identity, invalid volume domains, incompatible content
lineage, absent participant declarations, insufficient rollback capacity,
unsupported canonical contracts, or an unqualified requested authority backend
fail configuration explicitly. Moria does not substitute a default overworld,
hidden generator, all-empty world, implicit RNG seed, default participant
rollback strategy, or nondeterministic fallback.

### 4.2 Declare interest

**Requirement: REQ-009. Authority: C-004, D-009.**

The consumer declares bounded regions and the capabilities needed there.
Moria acknowledges the request, begins or prioritizes materialization, and
reports lifecycle changes. Readiness may arrive later and may differ between
authoritative matter and its derived presentation.

The consumer can change or withdraw interest. Withdrawal makes a region
eligible to become cold after in-flight work and persistence obligations are
safe; it does not immediately invalidate already issued results or discard
unsaved scars.

Interest alone never activates or deactivates the canonical simulation domain.
It may preload exact content for a future activation, but only a tick-stamped
canonical activation input changes membership. Camera position, per-client
render distance, local I/O timing, and render readiness cannot select what the
simulation processes.

### 4.3 Inspect

**Requirement: REQ-010. Authority: C-003, C-016, D-006, D-007.**

The consumer submits a bounded query against a stated world or volume scope.
The result identifies:

- the bounds and volumes actually inspected;
- the canonical tick and committed revision or revisions observed;
- the relevant canonical source hash when the result will drive deterministic
  behavior or verify replay;
- material and occupancy facts requested by the query;
- whether the result is complete, partial by explicit request, pending, or
  unavailable; and
- any lifecycle or budget condition preventing completion.

Moria never reports unloaded, failed, or unknown matter as empty. A query whose
bounds exceed the supported request contract is rejected rather than silently
clipped, unless the consumer explicitly asked for a partial result.

The product supports the inspection intents needed to consume material truth:

- sample matter at an addressed location;
- inspect a bounded three-dimensional region;
- determine occupancy in a point, region, or consumer collision shape;
- trace through matter to find ordered material encounters; and
- test overlap or swept movement against authoritative matter.

The exact primitive set, precision, and acceleration methods are selected in
technical design, but they must cover these intents without routing through a
render mesh.

Ordinary UI and diagnostic queries are observations and need not change the
canonical hash. If a query result will influence canonical state, it must be a
canonical tick-synchronous derivation or enter a later tick as an explicitly
encoded canonical input; asynchronous query arrival cannot become hidden
authority.

### 4.4 Mutate matter

**Requirement: REQ-011. Authority: C-005, AD-002, D-001, D-002.**

All changes after genesis enter through tick-stamped public commands. The core
design includes:

- remove or erode matter from a bounded target;
- place or replace material in a bounded target;
- apply a consumer-supplied bounded material patch or stamp;
- create or retire a material volume; and
- change the placement of a dynamic volume.

These are substrate effects, not game verbs. Moria does not price them, check
player inventory, infer damage, animate tools, or decide whether a game should
allow them.

Submission validates that the requested tick is eligible, the target and
material identities exist, the request is bounded and structurally valid,
required truth can be bound, and any supplied revision or hash precondition
still holds. The immediate outcome is either:

- **rejected**, with a stable reason and no effect; or
- **admitted for a named tick**, with a receipt for canonical ordering,
  transition, and publication.

An admitted command later reaches one terminal outcome:

- **applied**, with affected bounds and the new committed revision; or
- **failed**, with no committed effect and a reason the consumer can act on.

A single bounded matter-mutation command is one atomic public operation,
including when it affects multiple cells. All of its targeted changes become
visible together in that tick at one committed revision, or none of them do.
Admission and internal work may be staged, but queries, collision,
observations, rollback snapshots, persistence, and presentation never observe a
partially committed command. A consumer that wants independent success or
failure submits separate commands.

Pending edits are not visible as committed truth. Once applied, public queries,
collision truth, scars, observations, and eventual presentation all converge on
the same new revision. Consumers can attach correlation metadata so an
external system can match an observation to the request that caused it.

Revision preconditions let editors and behavior systems detect stale decisions
instead of overwriting newer matter. Moria reports a conflict; it does not
silently retry a command whose meaning may have changed.

Editor, administrative, test, behavior-adapter, volume lifecycle, restoration
helpers that alter state, and convenience paths receive no exception. Any
operation that creates a new canonical state is an ordered input to exactly one
tick. The rollback operation itself may reinstall an exact previously committed
frontier and replay its log; it may not manufacture altered state outside the
tick transition. An input received for an ineligible or already closed tick
receives a deterministic typed outcome defined by the versioned contract;
callback timing never decides whether it quietly slips into another tick.

### 4.5 Observe and react

**Requirement: REQ-012. Authority: C-003, D-002, D-006.**

Consumers and external behavior plug-ins can subscribe to bounded observations
for:

- committed matter changes and their affected bounds;
- volume creation, retirement, and placement changes;
- material-region lifecycle transitions;
- presentation becoming current or failing; and
- persistence checkpoint completion or failure.

Observation delivery is finite and backpressure-aware. If a consumer falls
behind far enough that events cannot be retained, Moria reports an explicit
gap and the last trustworthy revision. The consumer then obtains a bounded
snapshot and resumes. Silent event loss is forbidden.

Canonical outcome records identify their tick, deterministic within-tick
order, contract version, and resulting commitment. Observation delivery may be
asynchronous or coalesced, but delivery order cannot change simulation. A
consumer that wants an observation to affect later canonical state resubmits
the intended effect as an explicit canonical input.

An observing plug-in receives no privileged voxel ownership. If it wants to
change matter or move a volume, it submits an ordinary admitted command and
receives the same validation and failure behavior as any other consumer.

A GPU-oriented plug-in follows those same rules. Its bounded observation and
effect handoff may remain GPU-oriented where feasible, but admission and
terminal outcomes stay visible to the owning consumer through receipts,
revisions, observations, and errors. A faster path may not bypass validation,
make pending work appear committed, or introduce a second copy of world truth.

### 4.6 Present

**Requirement: REQ-013. Authority: C-011, D-007.**

Moria derives visible surfaces and optional dressing from committed matter at
a named canonical tick and revision.
Presentation supports both coherent organic volumes and crisp constructed
forms without making one material domain the universal aesthetic. Material or
volume presentation inputs may express different surface character; the
substrate is not limited to raw cubes, nor does it mandate smooth geology
everywhere.

Presentation has its own observable status:

- **absent** — no view is requested or available;
- **building** — a view for a committed matter revision is in progress;
- **current** — the view represents the reported current revision;
- **stale** — a valid older view exists while newer matter awaits rebuilding;
  or
- **failed** — view derivation failed while material truth remains intact.

A stale view may remain visible to avoid a needless visual hole, but it is
never used for collision or reported as current. Consumers can choose whether
their experience displays stale presentation, a diagnostic fallback, or
nothing. A mutation validation capture is complete only after presentation
reports the mutation's revision as current.

Rollback and replay do not require presentation for intermediate corrected
ticks. Existing derived work for invalidated future states is discarded or
ignored, and only the dirty regions of the final corrected canonical state are
scheduled for regeneration. Presentation contents, timing, mesh or vertex
order, lighting, LOD, and rendered pixels may differ between backends and do
not change canonical hashes or behavior.

The product distinguishes:

- **matter-backed assemblies**, such as vegetation, rocks, or constructed
  pieces that a consumer wants to query and mutate as material; and
- **derived dressing**, such as grass blades or surface scatter with no
  independent material identity.

Matter-backed assemblies register through the material-volume/object seam and
obey ordinary truth and mutation rules. Derived dressing remains anchored to a
specific matter revision and surface. When its supporting material changes,
the dressing is regenerated or removed; it cannot survive as a disconnected
prop that claims occupancy.

### 4.7 Persist and restore

**Requirement: REQ-014. Authority: C-009, D-002, D-005, D-006, D-008, D-009.**

The consumer requests a checkpoint at known committed revisions. A successful
checkpoint records substrate-owned scars and reconstruction state without
serializing derived geometry or requiring a raw dump of untouched homogeneous
volume. It also binds the canonical contract versions, content identity,
confirmed tick, canonical root commitment, allocator and RNG state, simulation
domain, and participant commitments needed to continue the same simulation.
Its completion identifies exactly which committed revisions and tick frontier
are durable. Mutations committed after those revisions remain dirty for a later
checkpoint; they are not silently included or lost.

Restore combines the consumer's compatible base content lineage with the saved
scars. It restores static and dynamic material edits, persistent volume
identity, and volume placement needed to reconstruct the same material truth.
Moria reports the restored revision context before dependent consumers resume.
If exact base reconstruction cannot be established, restore fails rather than
claiming that lineage compatibility alone proves equality.

Restore of a durable checkpoint and restore of a retained rollback frontier
both preserve canonical identity. Durable restore resumes only after all
required participants and contracts match. Rollback restore installs the named
retained frontier, restores or reconstructs coordinated participants, and
replays ordered inputs; it does not masquerade as a new mutation or traverse
untouched world matter.

The following are explicit restore failures, never implicit empty or best-effort
success:

- corrupt or incomplete scar data;
- an unsupported saved product contract;
- missing material identities;
- incompatible base content lineage; or
- a required volume or content source that cannot be reconstructed.

Migration, rebasing scars onto changed base content, and persistence of
external behavior state require explicit consumer action. Moria does not guess.

### 4.8 Shut down

**Requirement: REQ-015. Authority: C-003, D-002, D-005.**

Shutdown stops new admissions, allows the consumer to observe or cancel
outstanding noncommitted work according to the public contract, and reports
whether required persistence completed. Dirty material is not silently
discarded because a region became cold or the application began shutdown.
Confirmed canonical history is not advanced by shutdown timing. Any final
canonical state change must already belong to an eligible tick; cancellation
or abandonment of unconfirmed work is explicit and cannot corrupt a retained
frontier.

## 5. Lifecycle and consistency

### 5.1 Material-region lifecycle

**Requirement: REQ-016. Authority: C-004, D-009.**

For a bounded region of a volume, authoritative matter moves through these
consumer-visible states:

| State | Meaning |
| --- | --- |
| **cold** | No detailed authoritative representation is currently ready; base lineage and durable scars remain known. |
| **requested** | Consumer interest or pending work requires the region. |
| **materializing** | Base content and scars are being combined into authoritative matter. |
| **ready** | Queries, collision, and admitted mutation can use the reported revision. |
| **retiring** | Interest has ended and in-flight, observation, or persistence obligations are being resolved. |
| **failed** | Authoritative matter could not be made ready; the cause and retryability are reported. |

Lifecycle is not a promise about a particular storage tier. "Ready" describes
what the consumer may safely do, while CPU/GPU ownership and allocation remain
internal.

Presentation readiness is reported separately. A region can have ready truth
and building or failed presentation. It must never have presentation treated as
truth when authoritative matter is unavailable.

These readiness states are cache and access states unless a value within them
can influence a future canonical result. They do not by themselves add a
region to the simulation domain. Canonical simulation activation is the
separate tick-stamped lifecycle described by REQ-031; any lifecycle or
allocation state that can affect later transition bytes is included in the
canonical hash and rollback frontier.

### 5.2 Revision rules

**Requirement: REQ-017. Authority: C-003, C-005, D-001, D-002, D-006.**

- Queries describe committed revisions only.
- A mutation completion identifies the tick and one revision at which all
  effects of that command commit atomically.
- An observation for a change is emitted only after the containing tick
  commits.
- Collision results identify or are correlated with the committed matter
  revision, canonical tick, and source commitment used.
- Presentation and persistence identify the ticks and revisions they cover.
- A consumer may require a revision precondition for a command or a minimum
  revision for a query.
- Canonical events within a world and tick have one total order. No ordering is
  implied between independent worlds unless an explicit coordinated operation
  establishes it.
- Every confirmed tick produces one versioned canonical simulation hash.

These rules let a consumer build responsive experiences without treating
asynchronous completion as nondeterministic truth.

### 5.3 Residency and resource pressure

**Requirement: REQ-018. Authority: C-004, C-010, D-005, D-009.**

Consumers express bounded interest, priority, and needed capability; Moria
decides how to meet them within configured resource budgets. Under pressure it
may delay lower-priority materialization, retire eligible regions, or reject
new work. It must report which action occurred and why.

Moria may not:

- evict matter still required by an admitted operation;
- evict canonical state reachable from a retained rollback frontier or active
  replay;
- discard an unpersisted scar without explicit consumer authorization;
- return unknown matter as empty to satisfy a deadline; or
- make a derived view current by relabeling an older revision.

Telemetry lets a consumer understand active interest, lifecycle distribution,
authoritative residency, derived-view cost, queue pressure, and failed work
without exposing internal storage.

Resource pressure may prevent a tick from advancing according to a declared
deterministic failure policy; it may not make activation, allocation, input
ordering, or canonical output depend on which device operation happened to
finish first.

### 5.4 Canonical transition and publication

**Requirement: REQ-033. Authority: D-002, D-003, D-004.**

The transition path has one binding output discipline: no canonical byte is
selected, ordered, allocated, or produced by a race. For identical canonical
genesis and tick-batch bytes:

- canonical outcomes do not depend on submission, iteration, append,
  allocation, worker, dispatch, subgroup, or completion order;
- stable identities and output positions come from canonical state and keys,
  not a winning invocation;
- conflicts have a total, versioned resolution independent of arrival;
- parallel composition is legal only when every permitted ordering has the
  same exactly specified result; and
- canonical records are in their unique deterministic order before
  publication and hashing.

This is an observable outcome constraint, not a selection of sorting,
allocation, compaction, or synchronization algorithms. Technical design may
use parallel and atomic work only where every legal execution satisfies it.

A tick publishes its new state, canonical outcome records, participant
commitments, revision changes, snapshot frontier, and hash as one coordinated
confirmed result. No consumer or derived system observes an authoritative
mixture of old and new canonical state.

### 5.5 Incremental canonical commitment

**Requirement: REQ-034. Authority: D-006.**

Every confirmed tick publishes one canonical simulation hash without traversing
the entire world. Hash work follows changed canonical leaves and their affected
ancestors. Unchanged matter and unchanged volumes retain their existing
commitments.

The combined commitment proceeds conceptually from canonical bounded matter,
through stable volume and world identities, to coordinated participant
commitments. Physical storage position, insertion order, and unordered
iteration never affect combination order. Changing any voxel, placement,
identity or allocator state, RNG state, simulation-domain membership, or
participant commitment changes the appropriate hash path. Changing only a
derived cache does not.

Hash domain, encoding, and contract version are explicit. A change to any of
them creates a new replay and qualification identity rather than pretending
continuity with old hashes. The precise hash algorithm and hierarchy are
technical choices.

### 5.6 Rollback correction

**Requirement: REQ-035. Authority: D-005, D-006, D-008.**

A consumer requests correction to a retained confirmed tick and supplies the
canonical input sequence through the corrected present. Moria:

1. verifies that the target frontier, contracts, content, and participant
   commitments are retained and compatible;
2. installs that substrate frontier without copying or traversing untouched
   world matter;
3. restores or reconstructs every participant by its registered strategy;
4. replays tick batches in canonical order, comparing commitments when
   expected hashes are supplied; and
5. publishes the final corrected frontier or an explicit failure.

Rollback never exposes intermediate replay ticks as the new live present.
Derived caches do not run as prerequisites for intermediate ticks; invalidated
work is discarded, and the final corrected dirty set drives later
presentation. An active replay pins all reachable canonical state until it
finishes or fails.

Replaying the original input bytes must reproduce the original per-tick
canonical hashes and outcomes. Corrected inputs may produce a new sequence, but
that sequence receives the same atomic publication, hashing, participant, and
qualification guarantees.

## 6. Collision and dynamic-volume behavior

**Requirement: REQ-019. Authority: C-007, C-016, AD-005, D-003, D-007.**

Moria owns collision and occupancy **truth**, not motion policy.

Collision inspection:

- tests authoritative occupied matter for all relevant static and dynamic
  volumes;
- preserves volume identity, material identity, location, and surface/contact
  facts needed by a consumer;
- uses each volume's admitted placement for the tested revision;
- distinguishes no hit from unavailable truth; and
- remains valid when presentation is absent, stale, or rebuilt differently.

Collision authority used by a deterministic participant is a pure,
tick-synchronous derivation of canonical matter and placement. Identical
canonical inputs produce identical collision facts—or an identical canonical
collider artifact—across qualified backends and schedules. Simulation-facing
facts obey the canonical integer/fixed-point representation; float-valued
convenience results are derived observations and cannot feed back unnoticed.

The seam supports both consumer shapes without privileging either:

- a conforming GPU participant consumes bounded canonical occupancy or
  collision inputs without mandatory CPU readback; and
- a CPU or external participant consumes a canonically ordered collider
  artifact keyed by its source-state hash.

An artifact may be prepared and cached asynchronously. At its designated tick
it is either present and bound to the expected source hash, deterministically
unavailable under the declared fail-closed/no-advance policy, or replaced by
another explicitly canonical representation. "Not ready, therefore collision
is disabled" is never a valid canonical outcome.

Moria does not apply gravity, integrate velocity, separate overlapping bodies,
choose friction or restitution, deal damage, fracture matter, or convert matter
to debris. An external system may use collision facts to decide on motion or
edits, then request a volume placement or matter mutation through the normal
contract.

Movement and material editing are independent. A dynamic volume can move
without resampling its local matter, and it can be edited while retaining its
identity. Conflicting movement and edit requests use revisions and explicit
completion rather than a hidden privileged order. If they share a tick, their
canonical input order and transition contract determine the result; device
arrival order does not.

## 7. Content and presentation boundaries

**Requirement: REQ-020. Authority: C-011, C-012, C-013, D-002, D-009.**

Moria supports consumer-owned content without owning a content recipe.

The content seam must allow a consumer to provide:

- homogeneous empty and solid regions cheaply;
- arbitrary genuine three-dimensional matter, including voids and internal
  structure at any depth;
- material boundaries suitable for organic or constructed presentation;
- static and dynamic volumes;
- matter-backed assemblies; and
- authored, imported, stamped, or generated bounded content.

Natural geology—strata, caves, ore, aquifer bands, and buried structures—is a
useful validation content family because it proves deep volume and honest cut
faces. It is not a built-in generation pipeline or mandatory palette. A
fortress wall, sculpted object, or other consumer domain may demonstrate the
same substrate outcomes.

Moria may offer neutral registration and injection examples, but no example
becomes a default world law. In particular, there is no required continent,
biome, river, climate, vegetation, ruin-placement, or deterministic seed
algorithm.

Deterministic simulation does not promote generation into substrate identity.
A consumer-owned generator may participate in verified genesis construction.
After genesis, content entering the canonical simulation domain is named by
exact lineage or digest and activated by a tick-stamped input. Asynchronous
preloading may prepare those bytes, but absence or mismatch at activation fails
closed rather than substituting empty, newly generated, or alternate content.

## 8. Failure behavior

**Requirement: REQ-021. Authority: C-003, C-015, D-002, D-003, D-004, D-005, D-006, D-007, D-008, D-009.**

Failures are part of the public experience and must preserve truth.

| Condition | Required product behavior |
| --- | --- |
| Invalid configuration | World startup fails with actionable validation; no partial hidden world starts. |
| Content source is temporarily unavailable | The affected region remains pending or becomes failed according to the declared retry condition; it is not treated as empty. |
| Content source returns invalid content | The affected region becomes failed with the invalid scope identified; invalid matter is not admitted as truth. |
| Query is too large or outside the addressable domain | Reject with supported bounds; do not silently clip unless partial results were requested. |
| Query needs cold matter | Return pending/availability status or materialize under declared interest; never fabricate a result. |
| Mutation uses missing material or volume identity | Reject with no effect. |
| Mutation precondition is stale | Reject as conflict and report current revision context. |
| Canonical input targets an ineligible or closed tick | Return the versioned deterministic classification; never admit it according to callback arrival or silently move it. |
| Admitted mutation work cannot complete | Finish the receipt as failed with no portion of the command committed. |
| Other admitted work cannot complete | Finish the receipt as failed, identify whether any committed revision changed, and never leave success unreported. |
| Required canonical input or derivation is unavailable at transition | Apply the declared deterministic fail-closed or no-advance result; never substitute empty matter, disable collision, or publish a timing-selected state. |
| Canonical arithmetic overflows, exhausts range, or cannot represent an input | Produce the specified typed canonical outcome identically on every qualified backend; never rely on backend behavior. |
| Resource budget is exhausted | Defer, retire eligible work, or reject according to policy and expose pressure in telemetry. |
| Derived presentation fails | Keep authoritative matter usable, report failed presentation, and permit retry or diagnostic fallback. |
| Derived cache is late or corrupt | Ignore, discard, or rebuild it; canonical state, collision, simulation-domain membership, behavior, and hash remain unchanged. |
| Observation history is lost | Report an explicit gap and require a bounded resnapshot. |
| Persistence fails | Retain dirty truth where possible, report that it is not durably checkpointed, and block silent discard. |
| Save/base lineage is incompatible | Fail restore pending explicit migration or rebase; never replay scars speculatively. |
| Rollback frontier is outside the retained window | Reject correction with the available frontier range; never reconstruct from an unspecified state. |
| Retained rollback state is missing or corrupt | Fail restore without advancing; preserve the last trustworthy confirmed frontier and report the affected state. |
| Participant cannot restore or reproduce its commitment | Fail rollback or conformance explicitly; never accept divergence or silently remove the participant's contribution. |
| Activation content is missing or has the wrong identity | The activation tick follows its declared no-advance/fail-closed result; no empty or alternate content becomes active. |
| Replay hash diverges | Stop claiming identity at the earliest divergent tick and emit the portable divergence artifact. |
| Backend tuple lacks current qualification | Refuse deterministic-authority status for that tuple; local success or correct rendering does not upgrade it. |
| Qualified backend diverges | Mark the tuple unqualified for the affected canonical contract and retain byte-level evidence; do not average it into a warning. |
| External behavior plug-in fails | Moria truth remains valid; only effects canonically admitted through the normal tick contract may affect it. |

Errors identify their scope, retryability, and whether any committed revision
or confirmed tick changed. Deterministic failures are themselves canonically
classified when they occur in the transition path. Human-readable diagnostics
and machine-actionable categories describe the same condition.

## 9. Telemetry and diagnostics

**Requirement: REQ-022. Authority: C-015, D-002, D-004, D-005, D-006, D-008, D-009.**

Public telemetry makes scale and honesty reviewable without becoming a storage
escape hatch. It covers:

- world and volume readiness;
- interest and material-region lifecycle counts;
- authoritative and derived residency at a consumer-meaningful level;
- command admission, queueing, completion, conflict, and failure;
- query bounds, readiness, completion, and failure;
- current, stale, building, and failed presentation;
- revision lag between truth and presentation;
- checkpoint progress, durable revision coverage, and restore failure;
- observation backlog and gaps;
- behavior-extension boundary activity and consumer-meaningful transfer or
  readback pressure when that boundary is exercised;
- resource-pressure decisions;
- current, confirmed, oldest-retained, and replay target ticks;
- tick-batch size, admission outcomes, transition/replay work, and correction
  depth;
- changed canonical state and incremental hash work at a consumer-meaningful
  level;
- rollback storage, participant strategy costs, restore work, and
  restore-through-replay timing;
- simulation-domain membership and activation failures;
- per-tick canonical hashes and earliest-divergence context; and
- backend qualification identity, fixture and contract versions, evidence
  freshness, and local versus cross-GPU status.

Diagnostics may add raw-voxel, volume-boundary, lifecycle, revision, and
streaming visualizations. They are derived consumers of the public contract.
They do not gain mutation or storage privileges and are not product authority.

Measurements include machine and configuration context so evidence from
different systems can be compared without turning one machine's target into a
universal product requirement.

Non-authoritative telemetry may vary between runs and backends. It cannot feed
the transition path or alter canonical state. Canonical outcome records and
hashes remain separately versioned evidence rather than timing-sensitive
telemetry.

## 10. Validation experience

**Requirement: REQ-023. Authority: C-013, C-015.**

Moria is ready for review when it can prove its contracts through ordinary
consumers. The validation suite is scenario-based and fail-closed: missing
evidence is reported as not demonstrated, never inferred from a visually
plausible scene.

### Public-boundary proof

An external-style consumer configures a world, supplies content, declares
interest, queries, mutates, observes, checkpoints, and restores without an
internal path. The same scenario is reusable by in-repository harnesses.

### Truth-versus-view proof

With presentation present, absent, deliberately stale, and rebuilt, occupancy
and collision return the same matter facts for the same canonical tick and
revision. Derived geometry is discarded and regenerated without changing world
truth or its hash.

### Mutation honesty proof

The consumer submits tick-stamped removal and placement at exposed locations,
including across material boundaries and deep inside a volume. Completion
advances the tick and revisions;
queries and collision observe the edit; presentation becomes current with
honest cut and placed surfaces; derived dressing updates with its support.
Multi-cell commands are also forced to fail after admission in validation: no
targeted cell changes, no mutation revision commits, and no observer sees an
intermediate subset.

### Deep-volume proof

Consumer-supplied content contains reachable voids, bands, and structures
through substantial three-dimensional depth. Inspection and edits operate
throughout that depth. A heightmap with decorated underground surfaces cannot
pass.

### Sparse-scale and lifecycle proof

A content domain large enough that raw detailed residency would be
unreasonable remains bounded under interest changes. Homogeneous untouched
regions stay cheap, cold regions materialize on demand, and scars survive
retirement. Evidence reports resource use and lifecycle behavior.

### Persistence proof

A consumer checkpoints an edited static volume and an edited, moved dynamic
volume without saving derived presentation or dumping untouched homogeneous
matter. Checkpoint completion identifies its durable revisions, tick, canonical
commitment, and participant context while a later mutation remains explicitly
dirty. Restore against the reconstructable base lineage reproduces the same
material, volume identities, placements, and canonical hash.
Missing, incompatible, or non-reconstructable base content fails restore
without speculative scar replay.

### Dynamic-volume proof

A material volume is queried and collided with, moved through an admitted
placement change, edited in local matter, checkpointed, and restored. It keeps
stable identity; no physics or damage model is required to demonstrate the
contract.

### Behavior-extension proof

An optional minimal external plug-in observes a committed change, inspects
bounded truth, and submits an ordinary mutation or movement request. It owns
the reason and all behavior state. Removing the plug-in removes the behavior
without removing or changing Moria's material vocabulary.

When this proof exercises a GPU-oriented behavior engine, it also shows that
bounded observation and effect handoff can remain GPU-oriented where feasible
without changing admission, atomic completion, revision, or failure semantics.
A CPU-oriented proof can establish semantic compatibility, but by itself does
not evidence the intended GPU-performance path.

### Canonical transition proof

**Requirement: REQ-036. Authority: D-002, D-003, D-004.**

Every route capable of creating a new canonical state—including editor,
administrative, restoration-helper, test, and behavior-adapter routes—is shown
to produce a tick-stamped canonical input or to fail. Exact retained-frontier
restore is shown only to reinstall committed state and replay canonical inputs.
Genesis is the only pre-tick construction boundary.

The same genesis and tick-batch bytes run repeatedly with deliberately varied
submission threads, worker counts, insertion orders, dispatch/workgroup order,
completion schedules, and timing perturbations. Canonical bytes, identities,
outcome and rejection classifications, revisions, and hashes remain identical.
Collision-heavy allocation and output compaction preserve stable identity and
order. Overflow, exhaustion, and unrepresentable inputs produce the same typed
outcomes. Evidence demonstrates that no float-tainted, order-tainted, or
environment-tainted value reaches canonical publication.

### Snapshot, participant, and rollback proof

**Requirement: REQ-037. Authority: D-005, D-008.**

Validation retains at least 20 confirmed ticks while disjoint and overlapping
matter regions, placements, allocator/RNG state, simulation membership, and
participant contributions change. Multiple retained frontiers are restored and
replayed to the original present, matching every original per-tick commitment.

Evidence shows that restore installs the retained frontier without traversing
or copying untouched world material, state remains live while reachable by the
window or replay, and it becomes reclaimable only after it is no longer
reachable. Every registered participant strategy is exercised, including
declared resource exhaustion, missing-state, reconstruction, and divergence
failure.

### Incremental hash and replay proof

**Requirement: REQ-038. Authority: D-006.**

A bounded one-region matter change recomputes only its canonical leaf and
affected ancestors; unchanged volumes retain their commitments. Physical slot,
insertion, and map iteration order do not change the root. Targeted changes to
matter, placement, identities/allocators, RNG, simulation domain, and
participant state each change the appropriate commitment, while deliberately
changing derived render caches does not.

Fresh execution, replay, and rollback followed by replay reproduce the same
per-tick hash sequence. Poisoning one event or canonical state byte diverges at
the first affected tick and produces an artifact containing the evidence
specified by REQ-032.

### Cross-GPU conformance proof

**Requirement: REQ-039. Authority: C-010, C-015, AD-003, D-004.**

One retained conformance fixture runs byte-identical genesis and tick batches
on every tuple claimed as qualified. Evidence names GPU vendor and device,
driver version, backend, canonical contract version, and fixture digest. An
independent observer compares retained canonical records and hashes
byte-for-byte at every tick rather than trusting a self-reported pass flag.

Local repeatability, two machines using the same vendor, correct rendering, and
performance results are useful evidence but do not substitute for cross-vendor
conformance. A divergent or stale tuple is unqualified. Any change to a
transition-path kernel, canonical encoding, hash domain, or other qualified
canonical mechanism invalidates affected evidence until the permanent
GPU-capable gate reruns.

General hosted CI may validate non-hardware contracts, but it cannot claim
cross-GPU qualification without running on the declared hardware tuples.

### Simulation/presentation and domain-isolation proof

**Requirement: REQ-040. Authority: D-007, D-009.**

Missing, delayed, stale, differently ordered, or deliberately corrupt derived
presentation cannot change canonical collision, state, behavior, domain
membership, or hashes. Rollback requires no intermediate remeshing; the final
corrected dirty set regenerates presentation from corrected state.

Different I/O, preload, and render completion schedules produce the same
simulation-domain state and hash sequence. Missing or mismatched activation
content fails closed; overlapping activity proposals become one deterministic
union; and per-client camera or render interest cannot activate canonical
simulation.

### Failure proof

Validation deliberately exercises cold queries, invalid bounds, stale
preconditions, content-source failure, presentation failure, observation gaps,
resource pressure, incompatible persistence lineage, closed-tick inputs,
canonical arithmetic exhaustion, unavailable collision, failed activation,
participant restore failure, out-of-window rollback, replay divergence, and an
unqualified backend. Each fails in the documented state without becoming empty
matter, losing a scar, exposing storage, or advancing a timing-selected tick.

### Rollback performance qualification

**Requirement: REQ-041. Authority: C-015, D-005.**

Performance evidence measures the complete player-visible correction—restore
the selected confirmed frontier and replay through the corrected present—not
restore alone. The declared reference fixture is adversarial: multiple
independently controlled dynamic volumes and external behavior participants
form one interacting constraint chain while canonical matter changes at a
declared bounded rate. The fixture exercises both participant strategies. It is
a workload shape, not physics, grapple, vehicle, multiplayer, or other gameplay
added to Moria.

The technical validation plan declares its resident scale, population,
dirty-state rate and distribution, strategy mix, rollback depths,
simulation-frame interval, and hardware profiles before measurement. Hardware
that restores and replays 20 ticks within that frame interval qualifies for the
20-tick performance tier. Correct hardware that completes fewer ticks remains
deterministically correct but reports the lower measured rollback-per-frame
curve, allowing a consumer such as a netcode layer to clamp rollback depth or
add delay. Moria never silently claims the higher tier.

### Quality evidence

Benchmarks record mutation-to-commit time, commit-to-current-presentation time,
query responsiveness, lifecycle transitions, authoritative and derived
residency, checkpoint/restore behavior, and collision-truth agreement with
machine context. When the behavior-extension boundary is exercised, evidence
also reports its handoff latency and consumer-meaningful transfer or readback
pressure. Correctness scenarios must pass before such measurements support an
optimization claim. Target thresholds and curated routes belong to the
validation plan and technical design; this product design requires the
measures and honest status, not seed-specific numbers. Determinism evidence
additionally records tick transition and replay time, rollback storage,
incremental hash work, participant costs, correction-depth curves, and
qualification identity.

A walkable third-person scene may make several proofs easy for a human to see,
but its character, controls, camera, route, palette, assets, generator, and
performance target remain harness content. No walkable harness is required to
declare the substrate itself complete.

## 11. Scope reconciliation

**Requirement: REQ-024. Authority: C-001, C-002, C-003, C-004, C-005, C-006, C-007, C-008, C-009, C-010, C-011, C-012, C-013, C-014, C-015, C-016, AD-001, AD-002, AD-003, AD-004, AD-005, AD-006, AD-007, D-002, D-003, D-004, D-005, D-006, D-007, D-008, D-009.**

The subordinate design sources contain useful ideas at mixed levels of
authority. This design resolves them as follows.

### Selected into the product design

- Cheap representation of homogeneous empty and solid volume, sparse detail,
  and interest-driven materialization are required outcomes.
- A command/query/observation boundary hides GPU-resident storage and
  asynchronous completion from consumers without hiding state.
- GPU residency is selected for performance. The generic extension experience
  supports downstream behavior engines that keep their work primarily on the
  GPU when feasible, without giving them voxel ownership or importing their
  behavior into Moria.
- Correctness and explicit failure precede optimization; measured optimization
  continues behind invariant public semantics.
- Mandatory canonical tick authority, deterministic total input ordering,
  canonical integer/fixed-point simulation semantics, and arbitrary
  three-dimensional canonical placement are product constraints.
- Local replay and cross-GPU determinism are separate fail-closed guarantees;
  qualification is claimed only for evidenced backend tuples.
- At least 20 confirmed ticks of bounded shared-state rollback, incremental
  per-tick hashes, replay artifacts, and earliest-divergence diagnosis are
  public product capabilities.
- Simulation-domain membership is canonical and tick-stamped, while
  render/inspection/materialization interest remains local and cannot drive
  simulation implicitly.
- External canonical participants declare snapshot or reconstruction rollback
  and contribute to coordinated commitments without transferring behavior
  ownership to Moria.
- Collision used by deterministic behavior is a canonical derivation that can
  serve GPU participants without mandatory CPU readback or CPU participants
  through a source-hash-bound artifact.
- Surface presentation, dressing, debug views, and collision visualizations are
  derived from matter revisions and are never authority.
- Presentation can express smooth organic surfaces and sharp constructed
  surfaces; raw voxel display remains a valid diagnostic, not the mandated
  look.
- Deep three-dimensional content and everywhere mutation are validated with
  honest cuts, not heightmap illusion.
- Collision and occupancy read material truth; motion and contact response are
  external.
- Interest may come from cameras, agents, tools, or background work, and may
  keep non-camera regions active.
- Base content plus sparse scars is the persistence experience; derived
  geometry is rebuilt.
- Matter-backed assemblies and derived surface dressing are separate,
  truth-preserving choices.
- Comparable benchmarks and human-visible proofs report missing evidence
  honestly.

### Excluded by the approved boundary

- Any built-in deterministic or procedural geology, biome, river, cave, ore,
  aquifer, vegetation, ruin, or other world-generation pipeline.
- A networking protocol, rollback-netcode implementation, prediction policy,
  peer authority model, transport, input-delay policy, or correction blending.
- A built-in deterministic physics solver or any constraint, grapple, vehicle,
  tether, force, stretch, breakage, or island semantics. The adversarial
  constraint-chain fixture is performance evidence only.
- Deterministic rendered pixels, presentation timing, meshes, vertex order,
  LOD, lighting, dressing, debug output, or non-authoritative telemetry.
- An unbounded rollback history, a full canonical CPU voxel mirror, or a
  deterministic-authority guarantee for a backend tuple lacking current
  conformance evidence.
- The Product One region, material palette, third-person controller, camera,
  debug keys, curated route, milestones, machine targets, and release/demo
  positioning as Moria requirements.
- Physics, damage, fracture, strength, gravity, force, health, resistance,
  contact response, rigid conversion, re-voxelization, and debris rules.
- Cellular automata for fire, wetness, growth, granular settling, fluid
  simulation, weather, seasons, or structural collapse.
- Building gameplay, blueprints as player UI, mechanisms, rooms, navigation,
  work orders, economy, agents, combat, AI, spells, gas pricing, System/LLM
  behavior, multiplayer services, and game scripting policy.
- Current delivery or specific validation of ships, stations, multi-deck
  freeform hulls, or their game fiction. They motivate volume-general
  contracts only.
- Web/wasm as a current target and any claim of a finished visual engine before
  feasibility and visual acceptance are demonstrated.

### Deliberately deferred to technical design

The product outcomes do not select:

- crate count or source layout;
- spatial indexing, brick dimensions, voxel resolution, allocation, palette,
  payload, aggregate, or coordinate encodings;
- CPU/GPU work ownership, kernels, atomics, buffers, synchronization, or
  readback mechanisms, provided canonical state remains GPU-resident-capable
  without a mandatory full CPU mirror;
- canonical storage widths, scales, ranges, overflow/division/rounding/shift
  rules, byte encoding, coordinate mechanisms, and orientation representation,
  provided they meet REQ-028;
- deterministic ordering, conflict-resolution, allocation, compaction,
  publication, snapshot-sharing, reclamation, and incremental-hash mechanisms;
- the hash algorithm and hierarchy, replay persistence format, and
  qualification evidence storage;
- exact deterministic-participant adapter shapes and snapshot/reconstruction
  resource mechanisms;
- simulation-domain granularity and whether the first honest implementation
  keeps the complete session-scale domain resident;
- direct rendering, surface nets, dual contouring, another surface technique,
  or a measured hybrid;
- LOD, distant presentation, dressing generation, or object acceleration
  technique;
- exact collision primitives or collision-search algorithms, provided the
  inspection intents and truth guarantees above are met;
- cache, streaming-ring, eviction, compression, journal, or persistence
  formats;
- graphics backend details and machine-specific optimization;
- rollback fixture scale, population, dirty rate, strategy mix, frame interval,
  hardware profiles, measured-curve format, and other acceptance thresholds;
  and
- benchmark scenes, milestone order, and task decomposition.

Technical design should retain alternatives long enough to measure them
against GPU residency, portability, bounded access, sparse scale, presentation
quality, mutation latency, canonical semantics, rollback cost, and
cross-backend conformance. No technical selection may add a privileged consumer
path, make completion timing authoritative, or elevate a game-specific behavior
into substrate policy.

## 12. Resolved human design decisions

**Requirement: REQ-025. Authority: C-005, D-001.**

**Multi-target mutation completion is atomic.** One bounded matter-mutation
command commits all targeted cells together at one revision or commits none of
them. Partial application is not a supported public outcome. This selects the
consumer-visible semantic only; the staging and coordination that realize it
remain technical-design concerns.

### Deterministic-simulation decisions

**Requirement: REQ-043. Authority: C-008, C-010, C-015, D-002, D-003, D-004, D-005, D-006, D-007, D-008, D-009.**

Human product-design review resolves these additional calls, whose full
consequences appear throughout this document:

- **D-002:** canonical numbered-tick authority is mandatory after genesis; no
  nondeterministic convenience mutation path exists.
- **D-003:** the authoritative path uses exactly specified integer/fixed-point,
  race-independent semantics and supports arbitrary canonical 3D orientation.
- **D-004:** local replay and qualified cross-GPU determinism are distinct
  fail-closed invariants.
- **D-005:** at least 20 confirmed ticks are retained for bounded rollback;
  restore avoids whole-world traversal, while the 20-tick timing claim is a
  measured performance tier.
- **D-006:** every confirmed tick has an incremental canonical hash, and replay
  plus earliest-divergence evidence is public product behavior.
- **D-007:** presentation/caches remain noncanonical, while behavior-facing
  collision is canonical and source-bound.
- **D-008:** each canonical participant declares exactly one rollback strategy
  and joins coordinated commitments without surrendering behavior ownership.
- **D-009:** simulation-domain lifecycle is tick-stamped canonical state,
  separate from local interest.
No human product-design question remains open. Rollback workload parameters,
initial hardware tuples as available, and canonical orientation representation
are resolved outcomes requiring TDD parameterization, not invitations to
revisit the product decisions.

## 13. Completion criteria

**Requirement: REQ-044. Authority: C-001, C-003, C-004, C-005, C-006, C-007, C-008, C-009, C-010, C-011, C-012, C-013, C-014, C-015, C-016, AD-002, AD-003, AD-005, AD-007, D-001, D-002, D-003, D-004, D-005, D-006, D-007, D-008, D-009.**

The product design is realized when a Rust/Bevy consumer can, through the
public facade alone:

1. supply non-heightmap three-dimensional content for static and dynamic
   volumes;
2. keep large homogeneous regions cheap and materialize bounded interest;
3. establish a versioned canonical genesis and inspect committed truth with
   explicit bounds, readiness, tick, revision, and source commitment;
4. remove, place, patch, create, retire, activate, and move matter through
   ordered tick-stamped inputs that commit atomically per command and publish
   independently of completion timing;
5. move and edit dynamic volumes without importing motion policy;
6. collide against material truth while presentation is independently rebuilt;
7. observe changes and recover explicitly from observation gaps;
8. derive coherent organic and constructed presentation plus truth-anchored
   dressing;
9. checkpoint and restore scars and canonical continuation state against
   compatible, reconstructable consumer-owned base content with explicit
   durable revision and tick coverage;
10. retain at least 20 confirmed ticks, restore a retained frontier without
    whole-world traversal, coordinate both participant strategies, and replay
    to matching original hashes;
11. publish an incremental canonical hash every confirmed tick and produce a
    replay/divergence artifact that reproduces or diagnoses the sequence;
12. attach an external behavior participant without privileged storage or a
    substrate-owned behavior vocabulary, while permitting canonical GPU
    collision/occupancy and effect paths without mandatory CPU readback where
    feasible;
13. keep canonical simulation-domain activation independent of camera,
    rendering, preload, and I/O completion;
14. reproduce canonical state and outcome bytes under perturbed local
    schedules and across every claimed qualified GPU tuple while allowing
    presentation to differ; and
15. measure and fail-close each claim, including adverse lifecycle,
    persistence, rollback, participant, activation, replay, arithmetic, and
    qualification cases.

Passing one curated demo or meeting one machine's performance table is not a
substitute for these contract outcomes.
