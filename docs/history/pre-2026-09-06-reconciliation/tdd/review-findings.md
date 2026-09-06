## Auditor Turn — 2026-07-29T15:47:43Z

Mode: continue

Responding to: none

### Prior Findings Status

No prior auditor findings exist for this review run.

### New Findings

1. **Participant rollback is not implementable or transactionally isolated.**
   `TECH-016` requires both participant strategies and retained participant
   state (`architecture.md:318-336`), and `TECH-029` says the GPU adapter has
   equivalent semantics to the CPU adapter (`interfaces.md:457-493`). However,
   `TECH-054` exposes only device preparation and per-tick `encode`; it has no
   snapshot production, retained-snapshot handle, restore/reconstruct entry
   point, or restoration completion state (`collision-presentation.md:102-147`).
   Separately, `TECH-048` restores participants while replay is private, then
   promises that failure keeps the original live frontier
   (`content-persistence.md:227-245`), but neither participant API provides a
   private/staged participant instance, commit/abort protocol, or a specified
   way to restore the participant back to the original live state after a
   failed correction. Pinning the original participant state is not sufficient
   if `restore` has already mutated the one external participant. Define one
   bounded lifecycle for CPU and GPU participant state that covers per-tick
   snapshot/reconstruction products, private correction contexts, restoration
   completion, atomic commit, abort, generation loss, and failure cleanup.

2. **Participant snapshots have contradictory durable ownership and no
   checkpoint handoff.** `TECH-016` says snapshot participants “durably own”
   their opaque bytes while Moria pins a handle (`architecture.md:330-336`);
   `TECH-044` places snapshot blob digests in Moria's checkpoint manifest
   (`content-persistence.md:119-133`); `TECH-046` loads those blobs during
   restore (`content-persistence.md:170-183`); but `TECH-045` traverses and
   writes only scar/metadata GPU blobs and reports no participant-snapshot
   coverage (`content-persistence.md:144-162`). The TDD does not select who
   supplies snapshot bytes to `CheckpointStore`, how their bytes/digest are
   bound to the pinned frontier, what permit and byte limits apply, or whether
   manifest commit waits for their durable completion. Specify that ownership
   and async state machine so a successful checkpoint actually contains every
   blob its manifest requires and failure cannot report a participant frontier
   durable when it is not.

3. **The main-world/render-world completion and publication bridge is
   missing.** `TECH-031` assigns public queues, receipts, and root metadata to
   the main world while all submission and mapping state lives only in
   `RenderApp` (`gpu-runtime.md:13-27`). `TECH-032` drives mapping/decoding in
   the render schedule (`gpu-runtime.md:33-57`), and `TECH-037` merely mentions
   a generation-tagged “receipt bridge” (`gpu-runtime.md:190-200`). No contract
   defines the bounded return transport, its owner, capacity/backpressure,
   frame/schedule point, duplicate/late completion handling, or where the
   exclusive `TECH-013` publication transaction updates the main-world root,
   receipt, replay log, participant commitments, and observations together.
   Extraction is one-way, so this cannot be left implicit. Select the concrete
   cross-world bridge and publication schedule, including shutdown and device-
   loss draining.

4. **Canonical collision arithmetic remains algorithmically underspecified.**
   `TECH-007` specifies scalar widths and generic rounding rules, but not the
   exact fixed-point quaternion-vector transform and reduction sequence
   (`architecture.md:63-95`). `TECH-051` then delegates canonical collision to
   “slab, closest-feature, and separating-axis tests” and names output formats
   without selecting formulas for inverse transforms, degeneracies, contact
   point/normal construction and normalization, SAT axis ordering, or
   intermediate rounding (`collision-presentation.md:33-47`). Algebraically
   equivalent fixed-point implementations can produce different boundary
   hits, normals, and Q0.32 times, so CPU, WGSL, replay, and cross-GPU parity
   are not derivable from this contract. Specify the versioned algorithms and
   every tie/degenerate/overflow rule used to produce canonical facts, then
   extend the oracle/edge fixtures to cover them.

5. **The normative facade omits two approved public-interface capabilities.**
   First, `REQ-017` permits a query minimum-revision condition
   (`design-document.md:703-720`), and `TECH-024` refers to
   `minimum_revision`, but the normative `QueryRequest`/`QueryFrontier` shapes
   have no such field or type (`interfaces.md:279-306,324-339`). Second,
   `REQ-011` requires bounded correlation metadata that lets an observation be
   matched to its request (`design-document.md:518-525`), while the normative
   input header and outcomes contain no correlation identity
   (`interfaces.md:151-204`). Add concrete bounded, noncanonical facade types,
   propagation/lifetime rules, and result/observation behavior. If correlation
   metadata is deliberately outside canonical `TickBatch` bytes, specify the
   stable canonical-order association and what replay does with it.

6. **The required post-admission atomic-failure proof uses an unavailable
   privileged seam.** `TECH-064` requires `moria-qualify` to inject a production
   GPU diagnostic failure after admission and before multi-brick publication
   (`validation.md:140-157`). Yet `TECH-001` forbids the qualification binary
   from using an internal or test-only mutation hook (`overview.md:60-70`),
   `TECH-020` exposes no fault input (`interfaces.md:176-179`), and no public
   adapter/configuration contract can cause that named diagnostic at the
   required transition phase. Select an ordinary public/candidate
   qualification mechanism or a naturally reachable admitted failure that
   exercises construction before publication, state its bounds and authority
   isolation, and ensure an external-style consumer can invoke it without
   private access.

7. **Canonical RNG authority is named but not specified.** `REQ-028` requires
   every canonical-affecting RNG to have an explicit algorithm, seed, complete
   state, hash representation, and rollback representation
   (`design-document.md:288-316`). The TDD introduces `RngStreamId`, hashes
   “declared participant RNG commitments,” checkpoints those commitments, and
   validates changing RNG commitment bytes, but no `TECH` contract says whether
   Moria owns any RNG, defines an algorithm/state wire format, or requires a
   participant descriptor/snapshot/reconstruction proof to bind algorithm,
   seed, and complete state. A 32-byte commitment alone does not define
   transition semantics or reconstruct state. Either explicitly declare that
   Moria provides no RNG and specify the participant contract that satisfies
   `REQ-028`, or select the complete versioned RNG contract and apply it through
   genesis, hashing, rollback, checkpoint, replay, and validation.

The mechanical provenance audit passed: all 69 `TECH-001` through `TECH-069`
IDs are unique and present; each has one `Implements:` line; every approved
requirement except the intentionally absent `REQ-042` is covered; no
unapproved requirement is linked; and `traceability.md` exactly mirrors the
normative links.

### Questions For Coder

1. For correction failure, is the intended participant model a staged
   participant instance/state token that can be atomically installed, or an
   in-place participant that must be explicitly restored to the pinned
   original frontier before the correction receipt can fail?
2. Are participant snapshot blobs intended to be copied into Moria's
   `CheckpointStore`, or durably retained in a participant-owned store and
   referenced by a separately verified durable locator? The current manifest
   and restore text imply the former, while `TECH-016` implies the latter.

### Answers To Coder

No coder questions were pending.

### Notes To Coder

The one-package/library-plus-binary simplification in `decisions.md` is applied
faithfully, and the Bevy feature-plugin/source organization is concrete. The
findings above do not request a return to a workspace or extra crates.

### Approval

Not approved.

## Coder Turn — 2026-07-29T16:04:46Z

Mode: continue

Responding to: 2026-07-29T15:47:43Z

### Responses To Findings

1. Addressed. `TECH-016` now requires immutable participant state tokens and
   forbids adapter-global mutable canonical state. Tick preparation, snapshot
   restore, and reconstruction produce uninstalled tokens; one
   `FrontierBundle` swap installs substrate and participant state together.
   Corrections use a private context, success installs only the final bundle,
   and failure drains/drops staged CPU/GPU tokens without touching the pinned
   live bundle. `TECH-029` and `TECH-054` now expose matching CPU and GPU
   genesis, tick, snapshot restore, reconstruction, snapshot export, generation
   loss, and abort lifecycles.
2. Addressed. Moria now copies every `PerTickSnapshot` participant's verified
   bytes into the consumer's `CheckpointStore`; durable external locators are
   not accepted. `TECH-045` defines export reservation, source binding,
   length/digest verification, the bounded async state machine, shared staging
   and byte limits, manifest wait condition, and failure cleanup.
3. Addressed. `TECH-031` selects a shared `Arc<RenderCompletionBridge>` with a
   preallocated 32-cell fixed ring and per-job reservation. `TECH-032` defines
   render completion enqueue, main-world `First` draining, and one exclusive
   publication system that updates the root, receipt, rollback deque, replay
   log, participant tokens/commitments, revisions, and observations together.
   `TECH-037` defines duplicate, late-generation, shutdown, and device-loss
   draining.
4. Addressed. `TECH-007` now fixes quaternion Hamilton composition,
   normalization, Q2.28 matrix construction, vector/inverse transform order,
   intermediate widths, and rounding. `TECH-051` selects
   `moria-collision-v1` with exact cell boundary rules, sphere/capsule
   closest-feature formulas, SAT axes and depth selection, slab/continuous-SAT
   intervals, capsule active-set sweep, TOI conversion, contact witness/normal
   construction, degeneracy ordering, and overflow outcomes. `TECH-059`
   enumerates matching oracle and edge fixtures.
5. Addressed. `TECH-023` adds bounded per-volume minimum revision conditions
   and wait/stale policy to `QueryRequest`; `TECH-024` defines latest versus
   retained-frontier behavior. `TECH-019`/`TECH-020` add a bounded
   `CorrelationMetadata` sidecar, stable association after canonical sorting,
   result/observation propagation, byte accounting, expiry/gap behavior, and
   explicit exclusion from canonical bytes and replay identity.
6. Addressed. `TECH-040` now exposes one bounded public candidate-only
   diagnostic plan with the exact
   `AfterBrickConstructionBeforePublication` stage. It sets the ordinary
   production diagnostic record and follows normal `FailedNoAdvance` cleanup;
   it cannot mutate storage or run in authority mode. `TECH-064` invokes that
   public policy from the external-style qualification binary.
7. Addressed. `TECH-016` explicitly states that Moria owns no RNG and requests
   no entropy. A canonical-affecting participant RNG must declare algorithm
   identity/version/contract digest, state schema, seed bytes, and stream ID;
   each frontier commits complete state length/digest, snapshot participants
   include complete state bytes, and reconstructible participants reproduce
   them from genesis plus log. `TECH-029`, checkpoint, restore, replay, hash,
   and validation contracts now carry those descriptors and commitments.

### Answers To Auditor

1. The selected correction model is staged participant state. Participant
   operations return immutable uninstalled state tokens; Moria atomically
   installs the final set with the substrate root by swapping one
   `FrontierBundle`. No in-place restore or restore-back path exists.
2. Snapshot bytes are copied into Moria's configured `CheckpointStore`. Moria
   verifies and stores the bytes under their digest and does not commit the
   manifest until every participant blob is durable.

### TDD Changes Made

- Revised `architecture.md` for exact arithmetic/transforms, participant token
  publication, correction isolation, snapshot ownership, and RNG authority.
- Revised `interfaces.md` for correlation metadata, query revision floors, and
  the complete CPU participant lifecycle.
- Revised `gpu-runtime.md` for the bounded render-to-main bridge, exclusive
  main-world publication schedule, shutdown/device-loss draining, resource
  bounds, and public candidate diagnostics.
- Revised `collision-presentation.md` for the versioned canonical collision
  algorithms and equivalent staged GPU participant lifecycle.
- Revised `content-persistence.md` for participant snapshot storage,
  manifest gating, staged durable restore, and correction cleanup.
- Revised `validation.md` with bridge, participant, persistence, RNG,
  collision-edge, facade, and public fault-plan evidence obligations.
- Updated `overview.md` ownership wording and the exact `TECH-040` traceability
  link. Mechanical checks still find 69 unique `TECH` contracts, one
  `Implements:` line each, all 43 approved requirements covered, and exact
  traceability-table parity.

### Questions For Auditor

None.

### Notes To Auditor

The participant model preserves consumer ownership of state meaning and opaque
representation while giving Moria only the immutable handles and snapshot
bytes needed to coordinate publication, rollback, and durability. The public
candidate diagnostic is deliberately unavailable to qualified authority
worlds, so it provides the required external-consumer proof without creating a
canonical mutation path.

## Auditor Turn — 2026-07-29T16:13:31Z

Mode: continue

Responding to: 2026-07-29T16:04:46Z

### Prior Findings Status

1. **Participant rollback lifecycle — resolved.** `TECH-016` now makes
   participant tokens immutable and uninstalled until the one
   `FrontierBundle` swap, defines private correction contexts and abort/drain
   behavior, and rejects old-generation installation
   (`architecture.md:365-426`). `TECH-029` and `TECH-054` expose matching CPU
   and GPU genesis, tick, snapshot-restore, reconstruction, snapshot-export,
   and staged-token operations (`interfaces.md:509-599`;
   `collision-presentation.md:184-274`). `TECH-048` keeps the original bundle
   pinned and specifies cleanup without an in-place restore-back operation
   (`content-persistence.md:248-286`).

2. **Participant snapshot durable ownership — resolved for
   `PerTickSnapshot`.** `TECH-045` now reserves the declared export bytes,
   binds export to the pinned frontier, verifies length and digest, stores the
   bytes through `CheckpointStore`, waits for `BlobDurable` before manifest
   commit, and drains failures (`content-persistence.md:143-194`). The coder's
   answer that external durable locators are not accepted matches the TDD.

3. **Render-to-main publication bridge — resolved.** `TECH-031` selects the
   shared fixed 32-cell bridge and pre-reservation/backpressure rules
   (`gpu-runtime.md:10-48`); `TECH-032` selects main-world `First` draining and
   the exclusive publication critical section (`gpu-runtime.md:52-99`); and
   `TECH-037` covers acknowledgements, shutdown, and generation-loss draining
   (`gpu-runtime.md:237-262`).

4. **Canonical collision arithmetic — partially_resolved.** `TECH-007` and
   `TECH-051` now select concrete transform, closest-feature, SAT, sweep,
   reduction, witness, and tie algorithms, and `TECH-059` adds the matching
   edge fixtures. Two residual defects remain in New Findings 1 and 2: the
   selected quaternion normalizer can accept a non-unit result, and canonical
   collision facts still do not define the coordinate frame of their contact
   point and normal.

5. **Minimum-revision query and correlation facade — resolved.** `TECH-023`
   and `TECH-024` now define bounded per-volume revision floors and their
   `Wait`/`ReturnStale` behavior (`interfaces.md:306-387`). `TECH-019` and
   `TECH-020` define the bounded noncanonical correlation sidecar, canonical
   order association, byte accounting, propagation, expiry, gap, and replay
   behavior (`interfaces.md:112-230`).

6. **Post-admission atomic-failure proof — resolved.** `TECH-040` exposes a
   bounded public candidate-only one-shot diagnostic that enters the ordinary
   `FailedNoAdvance` validation/cleanup route and is forbidden in authority
   mode (`gpu-runtime.md:319-346`). `TECH-064` invokes it through the
   external-style public qualification consumer (`validation.md:159-187`).

7. **Canonical RNG authority — partially_resolved.** `TECH-016` now states
   that Moria owns no RNG and defines participant algorithm, seed, schema,
   complete-state commitment, snapshot, and reconstruction obligations
   (`architecture.md:428-453`); `TECH-029`, `TECH-044`, and `TECH-059` carry
   those descriptors into the facade, checkpoint manifest, and evidence.
   However, a reconstructible participant—including its RNG state—cannot
   actually be recreated from a durable checkpoint because the checkpoint
   stores replay digests/ranges but no replay bytes or verified durable replay
   locator. See New Finding 3.

### New Findings

1. **`TECH-007`'s selected quaternion normalization accepts values that are
   not unit quaternions.** The algorithm computes
   `norm = isqrt(x*x+y*y+z*z+w*w)`, then divides each component by that already
   truncated integer norm (`architecture.md:97-106`). For the accepted
   registration input `(x,y,z,w) = (1,1,0,0)`, `isqrt(2) == 1`, so the
   algorithm returns `(16384,16384,0,0)`; both components fit `i16`, but the
   result has length `sqrt(2)`, not one. Feeding it to the stated Q2.28 matrix
   produces a scale/shear rather than a rigid orientation and invalidates the
   maximum-radius displacement premise. Define either an accepted-input unit
   tolerance that rejects this case or a scale-preserving exact normalization
   algorithm that cannot lose the fractional norm before division, and add
   fixtures proving every accepted output satisfies the chosen unit/rigidity
   invariant and inverse/composition bounds.

2. **Canonical collision contact facts have no defined coordinate frame or
   final transform.** `TECH-051` explicitly transforms world shapes into
   volume-local space and performs narrow phase there
   (`collision-presentation.md:40-59`), then constructs witnesses, contact
   points, and normals from local cell/shape data
   (`collision-presentation.md:114-121`). `TECH-052` emits an unlabeled
   `contact point, normal` pair (`collision-presentation.md:143-151`) without
   saying whether either value is volume-local or world-space, naming its wire
   type, or specifying a local-to-world point/normal conversion for rotated
   dynamic volumes. CPU and WGSL implementations can therefore return
   different but superficially plausible canonical facts. Select the public
   coordinate frame and exact wire types. If facts are world-space, specify
   the TECH-007 conversion, normal rotation/renormalization, rounding, sign,
   and overflow rules; if local, label them as such and define the complete
   source placement data/consumer conversion contract. Extend rotated dynamic-
   volume parity fixtures to assert exact fact bytes.

3. **A durable checkpoint cannot reconstruct a
   `ReconstructibleFromCanonicalStateAndLog` participant.** `TECH-016` requires
   that strategy to reproduce state from canonical genesis/frontier plus log
   bytes (`architecture.md:424-426,447-449`), and `TECH-029::reconstruct`
   consumes a `ParticipantReplayLease` (`interfaces.md:533-537`). But
   `TECH-044` places only replay prefix/suffix *digests* in the checkpoint
   manifest (`content-persistence.md:111-134`), `TECH-045` says the participant
   contributes only its descriptor, commitment, and required replay range
   (`content-persistence.md:180-181`), and `TECH-046` loads only
   scar/node/snapshot blobs before asking it to reconstruct
   (`content-persistence.md:202-215`). A digest or range cannot supply the log
   after an application restart, and no durable replay-store/locator interface
   exists. Persist the exact bounded replay records as manifest-referenced,
   digest-verified blobs (including their admission, byte, async completion,
   and manifest gating), or select a verified consumer-owned durable replay
   source and define its restore failure contract. `TECH-049` must then count
   those bytes and may treat a checkpoint as a recovery anchor only after they
   are durable. Add restart-style restore tests with no surviving in-memory
   log, including a reconstructible RNG participant.

4. **The required participant failure policy is a name without a technical
   contract.** `REQ-030` requires every registration to declare bounded
   failure behavior. `TECH-029` includes
   `failure: ParticipantFailurePolicy` and says the descriptor fixes it
   (`interfaces.md:557-570`), but no TDD contract defines that type's closed
   variants, state transitions, effect on an attempted tick, retryability, or
   whether it can terminally fail a world. Other contracts independently say
   missing/divergent products or GPU adapter failures cause `NoAdvance`
   (`architecture.md:455-460`; `collision-presentation.md:266-274`), leaving it
   unclear what configurable behavior remains and whether a policy could skip
   a participant or otherwise weaken coordinated authority. Define the exact
   policy enum and behavior at genesis, ordinary preparation, rollback,
   restore, device loss, and shutdown; prohibit any variant that publishes
   without the registered participant; and add each policy row to the failure
   matrix.

The mechanical provenance audit still passes: all 69 `TECH-001` through
`TECH-069` IDs are unique; each has exactly one `Implements:` line; all 43
approved requirements are covered; no unapproved requirement is linked; and
the 337 normative requirement/contract pairs exactly match
`traceability.md`.

### Questions For Coder

1. For reconstructible participants, will the checkpoint own
   content-addressed replay-record blobs, or will restore require a separately
   registered durable replay source? Either can be coherent, but the current
   digest-only manifest is not reconstructible.
2. Are `TECH-052` contact points and normals intended to be world-space public
   facts or volume-local facts? Please make the choice explicit and carry its
   exact conversion through the wire and parity contracts.

### Answers To Coder

No coder questions were pending.

### Notes To Coder

The staged participant model, snapshot-store handoff, completion bridge,
query/correlation facade, and public candidate diagnostic match the prior
answers and are accepted. The remaining participant durability finding is
specific to the reconstructible strategy; it does not reopen the now-resolved
ownership choice for per-tick snapshot bytes.

### Approval

Not approved.

## Coder Turn — 2026-07-29T16:25:40Z

Mode: continue

Responding to: 2026-07-29T16:13:31Z

### Responses To Findings

1. Addressed. `TECH-007` no longer divides by a truncated integer square
   root. It now selects each rounded normalized component by exact squared
   comparisons in `i128`, enforces a proven quantized-unit shell, and uses a
   scale-independent rational quaternion rotation with the stored squared norm
   as denominator. This makes the pre-rounding transform orthogonal even when
   the Q1.14 vector is not exactly length 16,384. `TECH-059` now requires the
   `(1,1,0,0)` regression, generated shell/closure cases, rational
   orthogonality, transpose-inverse, and maximum-radius proofs.
2. Addressed. `TECH-051` and `TECH-052` now make public facts world-space,
   define `WorldContactPointQWire([i32; 3])` and
   `WorldContactNormalQWire([i16; 3])`, fix the normal direction from occupied
   cell toward query shape, and specify exact local-to-world point conversion,
   normal rotation, rounding, renormalization, sign preservation, and overflow
   failure. `TECH-059` and `TECH-062` require exact fact-byte parity for
   rotated and translated dynamic volumes.
3. Addressed. The checkpoint now owns exact content-addressed
   `moria-checkpoint-replay-v1` chunks covering the union of reconstructible
   participants' declared ranges. `TECH-044` defines their manifest entries;
   `TECH-045` defines pinning, admission, chunk bounds, byte reservation,
   verification, async blob states, and manifest gating; `TECH-046` loads and
   verifies them before reconstruction; and `TECH-049` counts them in all
   checkpoint and recovery-anchor limits. `TECH-066` now requires a
   restart-style restore after discarding the in-memory log, including exact
   reconstruction of a declared RNG stream.
4. Addressed. `TECH-029` now defines the closed, tagged policies
   `NoAdvanceExplicitRetry` and `FailWorld`, with exact outcomes for genesis,
   ordinary preparation, correction, durable restore, device loss, checkpoint
   export, and shutdown. No variant may skip a participant, reuse stale state,
   substitute an empty commitment, or fall back to another implementation.
   `TECH-028`, `TECH-038`, and `TECH-054` now carry the
   `RecoveringParticipant` and device-generation behavior consistently, and
   `TECH-060`/`TECH-065` require every policy/failure-site row.

### Answers To Auditor

1. The checkpoint owns content-addressed copies of every replay record needed
   by a reconstructible participant. A separately registered durable replay
   source is not part of this design.
2. Contact points and normals are world-space public facts. The local cell
   identity remains in each fact for provenance, while the named point and
   directed-normal wire fields use the exact final placement conversion.

### TDD Changes Made

- Revised `architecture.md` for exact square-root normalization, rational
  rigid rotation, durable replay-byte ownership, and policy-controlled
  participant failure.
- Revised `collision-presentation.md` for world-space fact wire types and
  conversion, plus GPU participant policy and generation-loss behavior.
- Revised `interfaces.md` for the closed participant failure-policy enum,
  lifecycle matrix, retry rules, and `RecoveringParticipant` state.
- Revised `content-persistence.md` for replay chunk format, manifest
  references, bounded asynchronous durability, restore verification, byte
  accounting, and recovery-anchor gating.
- Revised `gpu-runtime.md` so device recovery follows the participant policy
  and consumes durable snapshot/replay bytes without automatic retry loops.
- Revised `validation.md` with quaternion, rotated-contact, restart restore,
  reconstructible RNG, and complete participant-policy evidence.
- Updated `overview.md` to state Moria's ownership of copied checkpoint replay
  blobs.

### Questions For Auditor

None.

### Notes To Auditor

The provenance audit remains unchanged: 69 unique `TECH-001` through
`TECH-069` contracts, one `Implements:` line each, 337 normative
requirement/contract pairs, 43 approved requirements covered, and exact parity
with `traceability.md`. `git diff --check` passes.

## Auditor Turn — 2026-07-29T16:31:25Z

Mode: approved

Responding to: 2026-07-29T16:25:40Z

### Prior Findings Status

1. **Quaternion normalization and rigid placement — resolved.** `TECH-007`
   now selects component rounding by exact squared comparisons rather than a
   truncated integer norm, rejects values outside the stated quantized-unit
   shell, and applies the scale-independent rational quaternion rotation with
   the stored squared norm as denominator (`architecture.md:90-153`). The
   `(1,1,0,0)` regression is explicit, and `TECH-059` requires generated shell,
   composition, orthogonality, transpose-inverse, and displacement evidence
   (`validation.md:20-30`).

2. **Canonical collision fact frame and conversion — resolved.** `TECH-051`
   makes the public point and directed normal world-space and specifies the
   final local-to-world point transform, rational normal rotation,
   ties-to-even reduction, renormalization, zero/overflow failure, and
   direction preservation (`collision-presentation.md:117-141`). `TECH-052`
   names the exact Q23.8/Q1.14 wire types and fields
   (`collision-presentation.md:144-177`), while `TECH-059` and `TECH-062`
   require byte-exact rotated/translated-volume parity.

3. **Durable replay source for reconstructible participants — resolved.**
   `TECH-044` now puts each required replay range and content-addressed chunk
   descriptor in the manifest (`content-persistence.md:115-139`). `TECH-045`
   pins the exact confirmed records, bounds and verifies chunk construction,
   waits for every replay blob to become durable before manifest commit, and
   drains failures without publishing a checkpoint
   (`content-persistence.md:146-223`). `TECH-046` verifies continuity,
   digests, and prefix/suffix binding before providing the replay lease
   (`content-persistence.md:227-259`); `TECH-049` counts those bytes and gates
   recovery-anchor status (`content-persistence.md:323-353`). `TECH-066`
   requires restart-style restoration after discarding all in-memory log state,
   including exact RNG-state reconstruction (`validation.md:236-279`).

4. **Participant failure policy — resolved.** `TECH-029` defines the closed
   genesis-bound `NoAdvanceExplicitRetry` and `FailWorld` variants, their exact
   outcomes across genesis, ticks, correction, durable restore, device loss,
   checkpoint export, and shutdown, and forbids skip, stale-token reuse,
   fallback, empty commitments, or partial publication
   (`interfaces.md:562-627`). `TECH-028`, `TECH-038`, and `TECH-054` apply the
   recovery and device-generation behavior consistently, and `TECH-060` plus
   `TECH-065` require every policy/failure-site row to be exercised.

All seven findings from the first auditor turn remain resolved; the accepted
participant-token lifecycle, participant snapshot ownership, render-to-main
publication bridge, query/correlation facade, and public candidate diagnostic
were not regressed.

### New Findings

None.

The complete rereview found 69 unique stable `TECH-001` through `TECH-069`
contracts, exactly one `Implements:` line per contract, 337 unique normative
requirement/contract pairs, all 43 approved requirements covered, no invented
`REQ-042`, and exact pair parity with `traceability.md`. The latest committed
TDD diff passes `git diff --check`, and the worktree was clean before this
required ledger append.

### Questions For Coder

None.

### Answers To Coder

No coder questions were pending. The coder's two prior answers are accepted:
checkpoint-owned content-addressed replay records are now the selected durable
reconstruction source, and canonical contact points/normals are world-space
facts with exact final conversion.

### Notes To Coder

The TDD is implementation-ready. The selected one-package Bevy layout remains
faithful to the human simplification decision, the public qualification path
has no privileged storage access, and the participant/replay additions do not
introduce behavior ownership or game scope.

### Approval

Approved.

## Coder Turn — consumer-contract completeness revision

Mode: continue

Responding to: human-review feedback preserved verbatim in
`docs/tdd/decisions.md` under “Human review entry — consumer-contract
completeness”

### Responses To Findings

1. **Complete public facade — addressed.** New `TECH-070` is the normative
   facade closure. It connects construction/genesis/restore, tick reservation
   and submission, interest upsert/withdrawal, query submission, observation
   subscription/poll/resnapshot/resume/close, checkpoint, correction, recovery,
   telemetry, and shutdown to owned request/rejection and concrete receipt
   types. `TECH-021` supplies every receipt's callable `poll`/`cancel` shape,
   terminal retention, and family lifecycle.
2. **Complete `ResourceBudgets` — addressed.** `TECH-017` now defines the
   closed nested field schema, defaults, and portable maxima for identity and
   lifetime records, canonical work, content callbacks/residency, query and
   readback, observations, presentation, checkpoints, rollback/log/correction,
   participants, and runtime bridges/callbacks. `TECH-036` names every retained
   resource/overload outcome and eleven checked pre-genesis cross-limit rules.
3. **Base-content callback ownership — addressed.** `TECH-041` defines owned
   source/request descriptors and a non-clone `BaseBrickCompletion` token into
   a Moria-owned pre-reserved 2,048-byte sink. It specifies sequential exact
   writes, uniform output, bounded diagnostics, one invocation/completion,
   drop/cancel/panic/duplicate/late-generation behavior, validation, explicit
   retry, and permit release. `TECH-043` applies the same bounded completion
   discipline to checkpoint-store callbacks.
4. **Observation semantics — addressed.** `TECH-025` defines creation, finite
   admitted volume membership, kind/spatial/world-event filters, cursor start,
   count-and-byte ring bounds, poll limits, close/drop/shutdown, gaps, bounded
   resnapshot, and resume. Each ring record retains immutable append-time
   volume IDs and world bounds, including movement/retirement/create rules, so
   filtering never consults current placement or reclaimed directory state.
5. **Asynchronous lifecycle closure — addressed.** `TECH-021` defines
   admission ownership, pending phases, last cancellation point, terminal
   result, explicit retry, device-generation behavior, and shutdown behavior
   for genesis, ticks, interests, queries, observation subscriptions and
   resnapshots, checkpoints, corrections, restore, recovery, and shutdown. It
   also binds subordinate materialization, participant, presentation, and
   store jobs to their owning receipts.
6. **Participant simplification — addressed.** `TECH-011`, `TECH-016`,
   `TECH-029`, and `TECH-054` explicitly reject same-tick participant
   dependencies and retain no DAG, handoff, prior-feedback, or conflict
   subsystem. V1 supports bounded schema-bound opaque participant events for
   confirmed tick-receipt/replay delivery, but not the Moria-state observation
   ring. Effects use only ordinary `(ParticipantId, local_sequence)` phase-4
   ordering and preconditions. CPU/GPU participants still receive tick input
   and canonical artifacts, return state/effects/events, and keep their
   behavior vocabulary.
7. **Performance-policy replacement — addressed.** `TECH-068` explicitly
   supersedes `P1`–`P10`: universal correctness/boundedness/determinism remain
   gates, `TECH-067` is the one named 20-tick rollback tier, and all other
   performance evidence is hardware-contextual. Receipts cover sparse
   residency, materialization, mutation, query/collision, presentation,
   participants, checkpoints/restore, hashing, replay, and rollback.
8. **Normative snippet cleanup — addressed.** `MinimumVolumeRevision` has one
   `revision` field. `TECH-070` contains a mechanical public-type ownership
   index and requires every facade request/receipt to be callable. Validation
   now compiles the complete facade from the external-style binary, scans
   duplicate fields/type ownership, and exercises every budget/lifecycle row.

### Concrete TDD Evidence

- `TECH-070` is the only new stable ID; all 69 surviving IDs retain their
  meanings.
- The normative set has 70 unique `TECH` contracts and exactly one
  `Implements:` line per contract.
- All 43 approved requirements are covered; no `REQ-042` or unapproved
  requirement is cited.
- The 351 unique normative `REQ`/`TECH` pairs exactly match
  `traceability.md`.
- The normative Rust struct-field scan reports no duplicate field; the facade
  has no remaining `SubmitError` placeholder or unbounded `Vec` result shape.
- `TECH-060`, `TECH-064`, `TECH-065`, `TECH-068`, and `TECH-069` now require
  executable implementation-completeness evidence in addition to GDD
  coverage.
- `git diff --check` passes.

### Questions For Reviewer

None.

### Notes To Reviewer

The prior approval predates this human-requested completeness revision. A
rereview should state **GDD coverage** and **implementation completeness**
separately as required by `TECH-069`; neither conclusion is represented here
as a human decision or inferred from the other.

## Auditor Turn — 2026-07-29T17:22:40Z

Mode: continue

Responding to: consumer-contract completeness revision (the prior coder turn
has no ISO-8601 timestamp)

### Prior Findings Status

All findings resolved by the approved auditor turn at
`2026-07-29T16:31:25Z` remain resolved. The participant-token transaction,
snapshot and replay-blob durability, render-to-main publication bridge,
collision arithmetic/fact frame, query/correlation behavior, candidate
diagnostic, RNG contract, and participant failure policy were not regressed.

The eight items in the latest human-requested completeness revision have the
following status:

1. **Complete public facade — partially_resolved.** `TECH-070` now supplies
   callable interest, query, observation, checkpoint, correction, restore,
   recovery, telemetry, and shutdown methods (`interfaces.md:247-347`).
   However, the construction registries, concrete result/error shapes, and
   public replay/divergence route remain incomplete; see New Findings 1, 2,
   and 4.
2. **Complete `ResourceBudgets` — partially_resolved.** `TECH-017` defines a
   broad closed field schema and `TECH-036` supplies overload and cross-limit
   rules. The normative default values cannot pass their own rollback
   worst-case check; see New Finding 5.
3. **Base-content callback ownership — resolved.** `TECH-041` reserves the
   2,048-byte sink and lifetime record before invocation, makes the completion
   token non-clone, defines sequential exact writes and terminal cardinality,
   and covers cancellation, drop, panic, duplicate, late-generation, digest,
   and release behavior (`content-persistence.md:40-132`).
4. **Observation semantics — resolved.** `TECH-025` selects finite admitted
   volume membership, kind/spatial/world filtering, append-time immutable
   facts, honest count/byte gaps, bounded resnapshot, resume validation,
   close/drop, and shutdown behavior (`interfaces.md:851-1016`).
5. **Asynchronous lifecycle closure — partially_resolved.** `TECH-021` gives a
   useful admission/cancellation/retry/generation/shutdown matrix for its named
   families (`interfaces.md:535-684`), but a required replay operation is not
   in the matrix and several receipt results/errors are still undefined named
   types. See New Findings 2 and 4.
6. **Participant simplification — resolved.** `TECH-011`, `TECH-016`,
   `TECH-029`, and `TECH-054` consistently reject same-tick dependencies and
   retain a one-phase source-state/artifact to bounded effects/events/token
   model, with ordinary phase-4 conflict behavior and no DAG, handoff, or
   prior-feedback ABI.
7. **Performance-policy replacement — resolved.** `TECH-068` explicitly
   supersedes `P1`–`P10`, preserves universal correctness gates, retains only
   `TECH-067`'s named 20-tick tier, and reports all requested paths through
   hardware-contextual receipts (`validation.md:351-399`).
8. **Normative snippet cleanup — partially_resolved.** The duplicate
   `MinimumVolumeRevision.revision` field is gone, and the snippets have no
   duplicate struct fields. The claimed public-type closure is nevertheless
   false for material result, error, configuration, registry, and persistence
   callback types; see New Findings 1 through 3.

### New Findings

1. **The construction facade cannot register all identities and providers that
   genesis and later calls require.** `WorldBuilder` exposes only material,
   base-source, volume, and participant registration
   (`interfaces.md:171-181`). Yet a `GenesisVolume` refers to a
   `BaseAuthorityId` (`interfaces.md:225-232`), `BaseAuthority` may refer to a
   `ContentBlobStoreId` (`content-persistence.md:15-31`), canonical inputs
   require registered `InputSourceId`s, and the budgets explicitly count input
   sources and checkpoint stores (`interfaces.md:47-61`). There is no callable
   registration for a base-authority descriptor, bundled-content store, input
   source, checkpoint store, or replay sink, and the undefined
   `PersistenceConfig` does not establish one. Consequently an external
   consumer cannot construct several states that the TDD and `REQ-008` require,
   and a `CheckpointRequest` does not select among the promised per-world
   stores. Define the exact bounded registration/configuration shapes, stable
   ID ownership, duplicate behavior, store selection, and builder-freeze
   validation, then include each in `TECH-070`'s public-type/facade closure
   tests.

2. **`TECH-070` still uses unresolved result, error, and configuration
   placeholders instead of the normative results requested by the human.**
   The receipt signatures name `GenesisReady`, `InterestApplied`,
   `QueryResult`, `CheckpointCommitted`, `RestoreReady`, `Recovered`,
   `ShutdownReport`, and their operation-specific error types
   (`interfaces.md:561-610`), but the TDD does not define their closed Rust
   records/enums. The type-index assertion that every ready payload is a
   bounded owned record containing fields “promised” elsewhere
   (`interfaces.md:377-390`) is not a definition. This is observable: for
   example `TECH-022` promises exact covered bounds for partial interest, but
   there is no `InterestApplied` shape that can carry them; query prose does
   not define the sample/region/hit result variants; and shutdown prose does
   not define its bounded abandoned-receipt and dirty-root fields. Likewise
   `CanonicalContract`, `RollbackConfig`, `PersistenceConfig`,
   `PresentationConfig`, `TickReservation`, and `ParticipantRegistration` are
   named as closed public inputs without field or variant definitions. Replace
   the index-level placeholders with concrete bounded records/enums, including
   ownership and stable error variants, so the external-style compile test can
   validate semantics rather than merely accept opaque names.

3. **The checkpoint store cannot load a manifest, and its completion tokens
   have no callable completion API.** The normative `CheckpointStore` trait can
   `put_blob`, `get_blob` by `BlobDigest`, and `commit_manifest` by
   `CheckpointKey` (`content-persistence.md:167-177`). Durable restore begins by
   loading the manifest from a consumer-supplied `CheckpointKey`
   (`content-persistence.md:334-358`), but no method maps that key to manifest
   bytes; a caller cannot know a manifest blob digest before loading the
   manifest. In addition, `StoreSink`, `LoadSink`, and `CommitSink` are only
   described as following the base-completion discipline
   (`content-persistence.md:184-194`); no `write`, success, failure, drop, or
   disposition signatures make the trait implementable. Add a bounded
   key-based manifest load operation (with its atomic visibility semantics),
   exact sink method shapes and error/disposition types, and their
   cancellation/generation lifecycle. Connect the selected registered store to
   checkpoint, shutdown checkpoint, restore, and recovery requests.

4. **The approved public replay/divergence capability has no callable or
   internally coherent path.** `TECH-070` declares its method list to be the
   complete v1 facade but contains no replay-record export/sink, replay request,
   replay receipt, or divergence-artifact result (`interfaces.md:247-347`).
   `TECH-049` nevertheless requires the consumer to attach a replay sink
   (`content-persistence.md:490-517`), and `TECH-047` says replay uses
   `submit_tick` with an otherwise undefined “replay permit”
   (`content-persistence.md:390-407`). More seriously, it says a hash mismatch
   stops before publishing the divergent candidate
   (`content-persistence.md:416-421`), while the ordinary `submit_tick` request
   carries no expected hash that the exclusive pre-publication step could
   compare. `REQ-032`, D-006, and completion criterion 11 require replay and
   earliest-divergence evidence as public behavior, not a private qualification
   routine. Select a bounded public design: either a dedicated replay
   request/source/sink/receipt family or an explicitly defined correction-like
   private replay operation. Define record export retention/backpressure,
   expected-hash admission and comparison before publication, cancellation,
   divergence-artifact ownership/bounds, and lifecycle/validation rows.

5. **The normative default budgets fail `TECH-036`'s own required genesis
   inequality.** Defaults allow 16,384 changed bricks per tick with a 2 GiB
   rollback-retained budget and a 2 GiB authoritative GPU budget
   (`interfaces.md:64-85,134-143`). A dense changed brick costs 2,048 bytes and
   each independent scar path can copy 26 1,024-byte radix nodes
   (`architecture.md:53-57,230-239`; `gpu-runtime.md:112-120`). The explicitly
   required 20-frontier worst case is therefore up to
   `20 * 16,384 * (2,048 + 26 * 1,024) = 9,395,240,960` bytes (8.75 GiB) before
   volume/world metadata, participant state, and root tables. Distributing one
   changed brick across many of the default 65,536 volumes makes the paths
   independent, so common-prefix sharing does not rescue the declared worst
   case. `TECH-036` requires that exact worst case to fit both budgets before
   genesis (`gpu-runtime.md:271-290`), making the advertised defaults
   self-rejecting. Choose mutually consistent defaults/limits or a tighter
   proven allocation bound, and add an explicit test that the complete default
   configuration reaches genesis on a qualifying baseline adapter.

The mechanical provenance audit passes: the normative set has 70 unique
`TECH-001` through `TECH-070` headings, exactly one `Implements:` line per
contract, 351 unique requirement/contract pairs, all 43 approved requirements,
no `REQ-042`, and exact pair parity with `traceability.md`. `git diff --check`
passes and the worktree was clean before this required ledger append.

### Questions For Coder

1. Will public replay be a dedicated private-world replay receipt, or will
   `CorrectionRequest` be extended into the one replay/divergence operation?
   Either is viable, but ordinary live `submit_tick` cannot satisfy the stated
   pre-publication mismatch rule.
2. Is `CheckpointKey` intended to be a store-visible manifest locator? If so,
   please add the missing key-based manifest read and identify exactly how a
   world selects the registered store for checkpoint, restore, shutdown, and
   recovery.

### Answers To Coder

No coder questions were pending.

### Notes To Coder

The GDD traceability-coverage conclusion passes separately: every approved
requirement has semantically relevant technical representation and the
normative/link index is mechanically consistent. Implementation completeness
does not pass because the named consumer operations above cannot yet be
implemented or exercised through the claimed complete public contract. The
resolved compact one-package layout, one-phase participant model, and
correctness-first performance policy should be preserved while closing these
ordinary engineering gaps.

### Approval

Not approved.

## Coder Turn — 2026-07-29T17:48:26Z

Mode: continue

Responding to: 2026-07-29T17:22:40Z

### Responses To Findings

1. Addressed. `TECH-017` now exposes bounded registration for input sources,
   base authorities, bundled-content stores, checkpoint stores, replay sinks,
   and CPU/GPU participants in addition to the existing material, base-source,
   and volume calls. Provider descriptors own stable typed IDs; duplicates do
   not replace prior entries. Builder freeze validates every cross-reference,
   provider contract, per-provider bound, configured default checkpoint store,
   and mandatory replay sink before any callback or GPU allocation.
   `CheckpointRequest`, `RestoreRequest`, and `RecoveryRequest` carry exact
   `CheckpointStoreId`s, and shutdown already carries the exact checkpoint
   request.
2. Addressed. `TECH-017`, `TECH-018`, `TECH-020` through `TECH-029`, and
   `TECH-070` now define concrete closed configuration, registration, request,
   result, receipt, and error shapes. This includes `CanonicalContract`,
   `RollbackConfig`, `PersistenceConfig`, `PresentationConfig`,
   `TickReservation`, `ParticipantRegistration`, `GenesisReady`,
   `InterestApplied` with exact coverage, the complete query result variants,
   observation records/summaries, `CheckpointCommitted`, `RestoreReady`,
   `Recovered`, `ShutdownReport` with bounded abandoned/dirty fields, and one
   closed operation-error taxonomy. Fixed-width IDs, geometric values, bounded
   owners, canonical input payloads, participant callback values, and the
   coupled GPU participant names also have normative definitions rather than
   index-only promises.
3. Addressed. `TECH-043::CheckpointStore` now has
   `load_manifest(CheckpointKey, BlobLimits, ManifestLoadSink)`, where
   `CheckpointKey` is the store-visible locator rather than a blob digest.
   `StoreSink`, `LoadSink`, `ManifestLoadSink`, and `CommitSink` have exact
   sequential write/success/failure methods, closed error/disposition types,
   pre-reserved Moria-owned buffers, identity/length verification, and
   drop/cancel/duplicate/late-generation behavior. Manifest commit/load
   provides atomic whole-value visibility. Checkpoint, restore, shutdown, and
   recovery never fall through from the request's selected store.
4. Addressed. `TECH-047` now selects a dedicated public private-world replay
   family. A mandatory registered `ReplaySink` durably exports the genesis
   header and every confirmed record under bounded in-flight count/byte
   permits; log eviction waits for exact append completion and backpressures
   ticks instead of dropping records. `WorldBuilder::replay_records` consumes
   a bounded owned header/record sequence, reserves private roots,
   participants, results, and worst-case artifact bytes, and returns a
   pollable/cancellable `ReplayReceipt`. Expected root, outcome, participant,
   and event values are encoded in every record and compared before each
   private advance. Only complete success publishes the final world; earliest
   mismatch returns the bounded exact-prefix `DivergenceArtifact` and
   publishes nothing. `TECH-021`, `TECH-049`, and validation now include this
   lifecycle and pressure contract.
5. Addressed. The normative default for
   `canonical.changed_bricks_per_tick` is now 512 while retaining 16,384 as a
   portable configurable maximum. `TECH-036` gives the checked conservative
   no-sharing equation with explicit 256 MiB genesis and 2 MiB per-frontier
   metadata reserves. At all defaults, including the aggregate 64 MiB
   participant frontier allowance, the 20-frontier bound is exactly
   1,967,128,576 bytes and fits both 2 GiB byte budgets. Larger changed-brick,
   participant, or registry settings require correspondingly larger budgets.
   `TECH-060` adds a baseline default-genesis/20-frontier smoke fixture and a
   companion rejection for 16,384 changed bricks under unchanged 2 GiB
   budgets.

### Answers To Auditor

1. Public replay is a dedicated private-world `ReplayReceipt`, not an
   extension of live `submit_tick` or `CorrectionRequest`. It uses the same
   transition function in a private builder but compares each encoded expected
   value before advancing and publishes only the verified final bundle.
2. Yes. `CheckpointKey` is the store-visible atomic manifest locator.
   `CheckpointRequest`, `RestoreRequest`, and `RecoveryRequest` bind the exact
   registered store ID plus key; `ShutdownPersistence::RequireCheckpoint`
   embeds the same exact request. The configured default is used only by
   request-construction helpers and is never a failure fallback.

### TDD Changes Made

- Revised `architecture.md` with concrete stable ID, digest, cell, coordinate,
  placement, and geometry wire/value types.
- Revised `interfaces.md` with complete configuration/provider registries,
  result/error shapes, canonical input values, replay receipt/lifecycle,
  participant callback shapes, and corrected resource defaults.
- Revised `content-persistence.md` with bundled-store registration,
  key-addressed manifest loading, complete store completion tokens, exact
  store selection, checkpoint/restore results, mandatory replay export, public
  private-world replay, and bounded divergence artifacts.
- Revised `gpu-runtime.md` with store-bound recovery, replay resource
  accounting, a conservative 20-frontier allocation equation, and closed
  qualification types.
- Revised `collision-presentation.md` with concrete collision facts and named
  GPU participant adapter values.
- Revised `validation.md` with registry/store/sink tests, complete replay and
  divergence scenarios, exact-store isolation, and default-budget viability.
- Updated `overview.md` ownership/document-map language without changing the
  one-package architecture or scope boundary.

### Questions For Auditor

None.

### Notes To Auditor

No stable contract ID or `Implements:` link changed. Mechanical checks find 70
unique `TECH-001` through `TECH-070` contracts, exactly one `Implements:` line
per contract, all 43 approved requirements covered, no `REQ-042`, and exact
351-pair parity with `traceability.md`. The compact one-package layout,
one-phase participant model, staged frontier publication, and
correctness-first performance policy are preserved. `git diff --check` passes.

## Auditor Turn — 2026-07-29T17:57:35Z

Mode: continue

Responding to: 2026-07-29T17:48:26Z

### Prior Findings Status

1. **Construction facade and provider identities — partially_resolved.**
   `TECH-017` now has bounded registrations for every named provider, rejects
   duplicate IDs without replacement, validates cross-references at freeze,
   and puts an exact `CheckpointStoreId` on checkpoint, restore, recovery, and
   shutdown-checkpoint requests (`interfaces.md:219-294`;
   `content-persistence.md:388-434,514-573`; `gpu-runtime.md:380-434`).
   The configured replay export still lacks one required consumer-owned
   identity: no builder/configuration field supplies the
   consumer-selected `ReplayStreamKey`. See New Finding 2.
2. **Concrete public results, errors, and configuration —
   partially_resolved.** The previously missing records and closed error
   taxonomy now exist in `TECH-017`, `TECH-020` through `TECH-029`, and
   `TECH-070`. The facade nevertheless remains unusable from an external crate
   because its private bounded owners and participant/GPU leases have no
   normative construction or access methods. See New Finding 1.
3. **Manifest load and store completions — partially_resolved.**
   `CheckpointStore::load_manifest` and exact sink method names are now present
   (`content-persistence.md:193-295`), and request-to-store selection is
   explicit. The load completion rule requires an exact length that neither
   load call supplies or can know for the root manifest, so a conforming store
   cannot determine which short load is valid. See New Finding 3.
4. **Public replay/divergence capability — partially_resolved.** A dedicated
   private-world `ReplayReceipt`, owned request, pre-publication expected-value
   comparison, bounded divergence result, and validation scenarios now exist
   (`content-persistence.md:577-754`; `interfaces.md:455-458,1014,1050-1052,
   1110`; `validation.md:325-343`). Live export has no selected stream key and
   no terminal policy or callable redrive after a confirmed tick's sink append
   fails. See New Finding 2.
5. **Default rollback-budget viability — partially_resolved.** Reducing the
   default changed-brick count to 512 makes the advertised defaults fit 2 GiB
   even after the omitted participant-effect term is included. However,
   `TECH-036` labels its equation conservative and its validation fixture
   asserts an exact value that omits up to 4,096 participant placement effects
   per tick. See New Finding 4.

### New Findings

1. **The claimed complete public API still exposes opaque values that an
   external consumer or registered provider cannot construct or inspect.**
   `BoundedVec<T>`, `BoundedBytes`, and `OwnedBytes` have private storage
   (`interfaces.md:658-681`), but the TDD gives only constructor names in prose:
   it defines no signatures for inserting bytes/elements, reading a slice,
   iterating, obtaining a length, or consuming the value. Those types occur
   throughout constructible requests, participant descriptors and events,
   `ReplayRequest`, and the bytes passed *to* consumer stores. More critically,
   a CPU participant receives opaque `ParticipantStateLease`,
   `ParticipantReplayLease`, and `ColliderArtifactLease` values with no
   callable accessors even though the prose says it can inspect its prior
   state, exact replay, and collider bytes (`interfaces.md:1914-1985,
   2061-2072,2193-2203`). The GPU adapter likewise receives private borrowed
   input/state/effect/event/snapshot wrappers with no binding, metadata, or
   output methods (`collision-presentation.md:242-293`). Merely naming these
   wrappers does not make the public traits implementable, and the stated
   external-style compile/use test cannot populate a replay request or make a
   store read `OwnedBytes`. Define the minimal normative constructor/read/
   iteration APIs for bounded owners, the typed downcast or equivalent lease
   access for CPU state, the exact collider/replay views, and the coupled GPU
   binding/sink operations. Include their capacity, lifetime, error, and
   generation behavior in `TECH-070`/`TECH-060`.

2. **Live replay export has neither a selected stream identity nor a recovery
   policy after sink failure.** `ReplayStreamKey` is explicitly
   consumer-selected (`content-persistence.md:677-679`) and is mandatory in
   every `ReplaySinkRequest` and success echo
   (`content-persistence.md:601-620`), but `PersistenceConfig` contains only a
   `ReplaySinkId` (`interfaces.md:49-52`), provider registration supplies only
   the sink descriptor, and no other construction call accepts a stream key.
   Genesis therefore cannot form the required sequence-zero request without
   inventing or deriving consumer identity outside the contract. Separately,
   a confirmed tick remains pinned until a matching append success and later
   ticks eventually receive `PersistenceBackpressure`
   (`content-persistence.md:695-713,823-830`), but a failed append is terminal,
   store calls are not automatically retried, and the complete facade has no
   replay-export retry/redrive operation. The TDD does not say that such a
   failure fails the world either. Add the stream key to frozen consumer
   configuration with uniqueness/duplicate rules, and choose a bounded,
   observable append-failure transition: an explicit redrive operation or a
   stated terminal world policy. Define cancellation, shutdown, generation,
   receipt/observation/telemetry, and validation behavior for that choice.

3. **The load-sink success rule requires information absent from both load
   requests.** `get_blob` and `load_manifest` accept only an identity plus
   `BlobLimits { max_bytes }` (`content-persistence.md:194-203,220-222`), while
   `LoadSink::finish` and `ManifestLoadSink::finish` echo only the digest/key
   (`content-persistence.md:268-286`). The normative prose nevertheless says
   `finish` requires the “exact reserved byte length”
   (`content-persistence.md:303-312`). A maximum is not an expected length, and
   the initial key-based manifest read cannot know its encoded byte length
   before reading it. Requiring the cursor to equal `max_bytes` would reject
   every valid shorter value; accepting an arbitrary shorter cursor
   contradicts the claimed short-output check. Define whether expected length
   is an explicit request field when known, or whether finish accepts the
   actual `0..=max` cursor and validates digest/decode framing. Preserve
   unambiguous exact-length checks only where the caller actually has an
   expected length, then update the short/long fixtures accordingly.

4. **`TECH-036`'s revised “conservative” 20-frontier equation still
   undercounts canonical volume changes.** The equation uses
   `canonical.inputs_per_tick * 256` for changed-volume metadata
   (`gpu-runtime.md:281-306`). That covers the default 4,096 sealed inputs, but
   `TECH-011` adds a separate phase of participant effects
   (`architecture.md:318-354`), and `TECH-029` permits another independently
   bounded 4,096 effects per tick, including `SetPlacement`
   (`interfaces.md:188-190,2022-2049,2161-2169`). Thus a legal default tick can
   change up to 4,096 directly named volumes plus 4,096 more through
   participant placement effects. The current exact fixture is low by
   `20 * 4,096 * 256 = 20,971,520` bytes. The corrected default total still
   fits 2 GiB, so retain the simpler 512-brick default, but include all
   canonical effect sources in the checked formula and change
   `default-budget-smoke-v1` to assert the corrected complete bound.

### Questions For Coder

1. For a post-genesis replay-sink append failure, will the public contract
   expose a bounded explicit redrive, or will the world enter a stated failed
   state while retaining the record? Either can be coherent; indefinite
   pinning with no transition cannot.

### Answers To Coder

No coder questions were pending. I independently verified the two answers
given to the prior auditor: the TDD selects a dedicated private-world public
replay operation, and checkpoint/restore/recovery/shutdown checkpoint requests
now carry exact registered store IDs with no fallback.

### Notes To Coder

The mechanical provenance claims pass independently: the seven normative TDD
files contain 70 unique `TECH-001` through `TECH-070` headings, exactly one
`Implements:` line per contract, 351 unique requirement/contract pairs, all 43
approved requirements, no normative `REQ-042`, and exact pair parity with
`traceability.md`. `git diff --check` passed and the worktree was clean before
this required ledger append. The compact package/plugin organization,
one-phase participant model, store-ID isolation, and dedicated private replay
direction remain sound; the remaining defects are callable-contract and bound
closure, not a request to expand product scope.

### Approval

Not approved.

## Coder Turn — 2026-07-29T18:12:50Z

Mode: continue

Responding to: 2026-07-29T17:57:35Z

### Responses To Findings

1. Addressed. `TECH-070` now gives `BoundedVec`, `BoundedBytes`,
   `BoundedBytes64`, `BoundedUtf8`, and `OwnedBytes` concrete fallible
   constructors, mutation/read/iteration/length/capacity/consume methods, and
   lossless rejected-value records. `TECH-029` exposes participant-only typed
   state downcast, token metadata, exact replay-record iteration, and a
   source-bound collider byte view, with explicit pin/cancellation/lifetime
   behavior. `TECH-054` now fixes a 192-byte group-zero I/O wire, its binding
   layout and offsets, callable primary `bind_io` methods, range/capacity
   accessors, status/output validation, attempt coupling, and generation-loss
   behavior.
2. Addressed. The consumer supplies one exact `ReplayStreamKey` to
   `MoriaClient::begin_world`; it is frozen in the builder and paired with the
   configured sink for sequence zero and every later append. `TECH-017`
   defines duplicate reservation and retired-stream tombstones. I selected the
   terminal policy for post-genesis append failure: the confirmed tick receipt
   remains valid, the world enters `Failed`, the exact undurable record remains
   pinned, no redrive/fallback exists, and a scoped lifecycle error, telemetry,
   and shutdown report expose the outcome. Device loss, late completion, and
   cancellation behavior are explicit.
3. Addressed. `BlobLimits` now carries `expected_bytes: Option<u64>`.
   Every digest-based `get_blob` call supplies the exact known length and
   rejects short or long output. Only initial key-based manifest load uses
   `None`; it accepts an actual cursor up to the maximum and then requires
   complete declared framing, checksum, and no trailing bytes. Manifest scar,
   snapshot, and replay descriptors now carry exact uncompressed lengths, and
   restore passes them to each load.
4. Addressed. `TECH-036` now counts
   `canonical.inputs_per_tick + participant.effects_per_tick` in the
   changed-volume term. The complete default bound is
   `1,988,100,096` bytes, including independent direct and participant
   placement changes, and `default-budget-smoke-v1` asserts that value with
   both 4,096-record sources exercised.

### Answers To Auditor

1. Post-genesis replay-sink append failure is terminal for the published world;
   v1 exposes no redrive operation. The already-confirmed frontier remains
   trustworthy and its tick receipt remains `Ready`, but new authority
   admission closes, the undurable record remains pinned, and shutdown is the
   only remaining release path.

### TDD Changes Made

- Revised `interfaces.md` for per-world replay stream construction, bounded
  owner APIs, participant state/replay/collider views, exact failure
  observation, and lease lifetimes.
- Revised `collision-presentation.md` for the fixed GPU participant bind-group
  ABI, public binding/range methods, status sinks, validation, and generation
  cleanup.
- Revised `content-persistence.md` for exact/unknown load lengths, manifest
  length descriptors, selected stream identity, and terminal append failure.
- Revised `gpu-runtime.md` and `validation.md` for the complete rollback bound
  and external-style API/load/replay-failure evidence.
- Mechanical checks still find 70 unique `TECH-001` through `TECH-070`
  contracts, one `Implements:` line each, 351 normative pairs, all 43 approved
  requirements, no `REQ-042`, and exact traceability parity.

### Questions For Auditor

None.

### Notes To Auditor

The replay stream key is per builder rather than global `MoriaConfig`, so one
`MoriaClient` can host multiple worlds on the same registered sink without
deriving identities; duplicate `(sink, stream)` pairs remain fail-closed. The
terminal append policy does not roll back an already-confirmed tick or pretend
the failed export became durable. `git diff --check` passes.

## Auditor Turn — 2026-07-29T18:18:52Z

Mode: continue

Responding to: 2026-07-29T18:12:50Z

### Prior Findings Status

1. **The public bounded-owner and participant lease surface —
   resolved.** `TECH-070` now defines fallible construction, mutation,
   inspection, iteration, length/capacity, and consuming operations for the
   bounded owners (`interfaces.md:672-783`). `TECH-029` provides typed
   participant-state downcast, exact replay-record iteration, and a
   source-bound collider view with pin and cancellation behavior
   (`interfaces.md:2088-2177,2343-2360`). `TECH-054` now exposes concrete GPU
   metadata/range/binding methods. The newly selected GPU method semantics
   still contain an implementation defect, recorded separately below.
2. **Live replay stream identity and append-failure policy —
   partially_resolved.** `begin_world` now accepts the consumer-selected stream
   key, the frozen `(sink, stream)` pair reaches every append, and the TDD
   explicitly chooses terminal `Ready -> Failed` behavior with no redrive
   (`interfaces.md:207-213,280-292,402-404`;
   `content-persistence.md:700-763`). That answers the prior consequential
   question. The promised shutdown evidence has no representable public field,
   and the new tombstone accounting contradicts the stated retry path; see New
   Findings 1 and 2.
3. **Manifest and known-length blob loads — resolved.** `BlobLimits` now
   distinguishes `Some(expected_bytes)` from the one initial unknown-length
   manifest lookup, exact-length blob completion is enforceable, and complete
   manifest framing/checksum/trailing-byte validation handles the unknown
   length (`content-persistence.md:193-223,299-337,555-570`). The matching
   negative fixtures are explicit (`validation.md:100-113`).
4. **Complete rollback allocation bound — resolved.** `TECH-036` counts both
   direct canonical inputs and participant effects and derives the corrected
   `1,988,100,096`-byte default bound (`gpu-runtime.md:275-318`). The default
   fixture exercises both independent 4,096-record sources
   (`validation.md:154-160`).

### New Findings

1. **The selected terminal replay failure cannot be represented by the
   promised shutdown result.** `TECH-047` says shutdown reports the exact
   pinned replay record as undurable before releasing it, and validation
   asserts that observable outcome (`content-persistence.md:737-757`;
   `validation.md:131-136`). However, `ShutdownReport` contains only the last
   confirmed frontier, checkpoint-oriented `last_durable`, abandoned facade
   receipt IDs, dirty scar/metadata summaries, and an optional checkpoint
   result (`interfaces.md:591-603`). It has no replay stream, sequence, tick
   range, record digest/bytes, or replay-durability status. `WorldLifecycleFact`
   likewise carries only state, device generation, and `OperationError`
   (`interfaces.md:1596-1600`); that error identifies the sink and committed
   frontier but not the failed append identity. Therefore the validation claim
   and the shutdown contract are not implementable as written. Add a closed,
   bounded replay-export failure/durability record to `ShutdownReport` (and to
   the lifecycle fact if that observation is meant to identify the append),
   including enough of the original `ReplaySinkRequest` identity to distinguish
   the undurable record. State its count/byte bound and release behavior. Also
   replace the nonexistent `FailureCounter::StoreFailure` name in
   `content-persistence.md:747` with the actual
   `FailureCounter { code: ErrorCode::StoreFailure, ... }` contract.

2. **Replay-stream tombstones consume the world budget and make the documented
   genesis retry impossible under the normative default.** `identity.worlds`
   defaults to one (`interfaces.md:74-76`), while any construction failure
   after the sequence-zero call leaves a client-lifetime tombstone that “counts
   against `identity.worlds`” (`interfaces.md:289-292`). The genesis lifecycle
   nevertheless promises that after an accepted failure the consumer can
   construct a new builder (`interfaces.md:1206-1209`). With the default
   budget, the tombstone permanently occupies the only world slot, so even a
   new builder using a different stream cannot follow that retry contract.
   Normal sequential shutdown/recreation also accumulates the same
   client-lifetime pressure without a separately named capacity. Give retired
   stream reservations their own finite client-level budget and overload
   outcome, or revise the accounting/lifecycle so a replacement builder is
   actually admissible while reused `(sink, stream)` pairs remain rejected.
   Add the accepted-genesis-failure/default-budget retry fixture, not only the
   simultaneous duplicate-pair fixture.

3. **`TECH-054` promises eager GPU pass/layout validation that the exposed wgpu
   API cannot perform.** `bind_io` receives only
   `&mut wgpu::ComputePass` (`collision-presentation.md:288-325`), but a wgpu
   compute pass exposes no query for the currently selected pipeline layout,
   bind-group compatibility, encoder identity, or device generation, and
   `set_bind_group` itself has no fallible return. A participant can also bind
   the group and select an incompatible pipeline afterward. Consequently the
   claim that `bind_io` validates the pass/layout and returns a participant
   error while encoding no dispatch is not implementable
   (`collision-presentation.md:396-406,433-436`). wgpu can report this mismatch
   later through the balanced validation scope, which is sufficient to prevent
   publication, but that is a different milestone and error path. Either make
   `bind_io` an infallible binding operation and specify that scoped
   post-encoding wgpu validation fails the attempt, or replace the raw pass with
   a Moria-owned encoder/pipeline abstraction capable of the promised eager
   checks. Update the external GPU fixture to assert the selected achievable
   behavior.

4. **Constrained public identity newtypes still have neither validating
   constructors nor coherent field visibility.** `TECH-005` specifies nonzero
   and range-constrained IDs and says public constructors validate those rules,
   but its only callable shape is a public-field tuple struct such as
   `MaterialId(pub u16)`, `ParticipantId(pub u32)`, and
   `InputSourceId(pub u32)` (`architecture.md:14-25,32-54`). Provider and
   persistence IDs repeat the same public-field pattern
   (`content-persistence.md:15-24`). If those fields remain public, tuple
   construction bypasses the claimed validation; if implementation makes them
   private under the latitude stated in `interfaces.md:3-5`, an external
   consumer has no normative constructor or raw-value accessor. `TECH-070`
   claims every public named type is closed and indexes these as fixed-width
   newtypes, but defines methods only for bounded collection/byte owners
   (`interfaces.md:672-805`). Define the minimal `try_from_raw`/`get` (and
   fixed-byte equivalents) for every constrained or opaque consumer-supplied
   newtype, with private fields where construction invariants matter, or
   explicitly make boundary validation—not construction—the contract. Extend
   the external compile/use fixture to construct valid IDs and reject each
   reserved/out-of-range value through the selected public path.

### Questions For Coder

None. The terminal replay policy choice is now explicit; the remaining issues
are ordinary contract-representability and API corrections.

### Answers To Coder

No coder questions were pending. I verified the claimed terminal choice:
post-genesis replay append failure preserves the already-confirmed frontier
and tick receipt, fails the world, and exposes no redrive or sink fallback.

### Notes To Coder

The revised bounded-owner, CPU participant, known-length load, and rollback
equation work is sound and should be preserved. Independent mechanical checks
still find 70 unique `TECH-001` through `TECH-070` headings, exactly one
`Implements:` line per contract, 351 unique normative pairs, all 43 approved
requirements, no `REQ-042`, and exact parity with `traceability.md`.
`git diff --check` passed and the worktree was clean before this required
ledger append.

### Approval

Not approved.

## Coder Turn — 2026-07-29T18:29:39Z

Mode: continue

Responding to: 2026-07-29T18:18:52Z

### Responses To Findings

1. Addressed. `TECH-047` now defines the closed `ReplayExportFailure` record
   with the exact sink and original stream/sequence/tick-range/count/length/
   digest identity plus failure code. The record appears in both the terminal
   `WorldLifecycleFact` and `ShutdownReport`, is bounded to one record and
   128 retained bytes by per-stream append serialization, and is constructed
   before shutdown releases the pinned raw record. The nonexistent
   `FailureCounter::StoreFailure` reference is replaced with the actual
   `FailureCounter { code: ErrorCode::StoreFailure, ... }` shape.
2. Addressed. `IdentityBudgets` now has a separate client-level
   `retired_replay_streams_per_client` pool (default 4, maximum 64).
   Genesis reserves its eventual tombstone before the sequence-zero callback;
   exhaustion returns `RetiredReplayStreamCapacity` without invoking the
   sink. Active/retired streams no longer consume `identity.worlds`, and an
   accepted genesis failure releases the default sole world permit while
   preserving only the failed pair's tombstone.
3. Addressed. Every GPU participant `bind_io` is now an infallible group-zero
   binding operation. The contract no longer claims that a raw
   `ComputePass` can expose or eagerly validate its selected pipeline/layout
   or encoder identity. Moria wraps the complete participant encoding in a
   balanced wgpu validation scope; incompatible pipeline selection before or
   after binding fails the attempt before any participant result or canonical
   frontier can publish.
4. Addressed. Constrained and opaque identity/key newtypes now have private
   fields and exact validating or lossless constructor/accessor families.
   Nonzero/high-bit-reserved IDs reject invalid raw values,
   `ReplayStreamKey` rejects the all-zero reserved value, fixed-byte keys,
   digests, lineages, and unconstrained counters preserve all bits, and
   Moria-created IDs expose inspection without public construction.
   `QuatQ14` also now has a private field and explicit normalized
   construction/composition/inverse API, removing the adjacent unchecked
   constrained-newtype ambiguity.

### TDD Changes Made

- Revised `architecture.md` with private identity/digest fields,
  `NewtypeValueError`, exact scalar/byte constructor and accessor contracts,
  and a callable validated `QuatQ14` surface.
- Revised `interfaces.md` with the retired-stream pool and overload outcome,
  replacement-genesis accounting, replay-failure fields on lifecycle and
  shutdown results, private consumer identifiers, and the complete
  public-type index.
- Revised `content-persistence.md` with provider/key constructors, the exact
  replay-export failure record, per-stream append serialization, telemetry
  shape, metadata bounds, and shutdown pin-release ordering.
- Revised `collision-presentation.md` to use achievable infallible wgpu
  binding plus scoped post-encoding validation.
- Revised `gpu-runtime.md` with the separate client-level replay-stream
  resource bound and pre-callback overload behavior.
- Revised `validation.md` with external identity construction/rejection,
  accepted-genesis-failure retry, tombstone exhaustion, exact replay-failure
  reporting, and incompatible-pipeline validation-scope fixtures.

### Questions For Auditor

None.

### Notes To Auditor

No stable `TECH` ID or `Implements:` link changed. Mechanical checks find 70
unique `TECH-001` through `TECH-070` contracts, exactly one `Implements:` line
per contract, 351 normative pairs, all 43 approved requirements, no
`REQ-042`, and exact traceability parity. `git diff --check` passes.

## Auditor Turn — 2026-07-29T18:37:43Z

Mode: continue

Responding to: 2026-07-29T18:29:39Z

### Prior Findings Status

1. **Terminal replay-append failure reporting — resolved.**
   `ReplayExportFailure` now retains the sink and complete original
   `ReplaySinkRequest` identity plus failure code
   (`content-persistence.md:634-648`). Both the terminal
   `WorldLifecycleFact` and `ShutdownReport` expose the record
   (`interfaces.md:616-623,1620-1625`), and `TECH-047` fixes its one-record,
   128-byte retention and raw-byte release ordering
   (`content-persistence.md:774-789`). The telemetry reference now uses the
   actual `FailureCounter { code: ErrorCode::StoreFailure, ... }` shape.
2. **Replay-stream tombstone accounting and default retry — resolved.**
   `IdentityBudgets` has a distinct client-lifetime retired-stream pool
   (`interfaces.md:74-89`). `TECH-017` reserves it before the sequence-zero
   callback, names the exhaustion error, separates it from the world permit,
   and permits a different-stream retry under the default single-world budget
   (`interfaces.md:281-308`). The validation fixture covers failure, retry,
   duplicate rejection, and pool exhaustion (`validation.md:137-154`).
3. **GPU participant eager pass/layout validation — resolved.**
   `TECH-054` now makes `bind_io` an infallible group-zero binding, explicitly
   disclaims queries that raw `wgpu::ComputePass` cannot perform, and assigns
   incompatible pipeline/layout use to a balanced post-encoding wgpu
   validation scope before publication
   (`collision-presentation.md:388-436`). The external GPU fixture asserts
   that achievable milestone (`validation.md:126-135`).
4. **Constrained public identity construction — partially_resolved.**
   The requested private fields and callable constructor/accessor families now
   exist for identities, digests, provider IDs, keys, lineages, and
   `QuatQ14` (`architecture.md:32-97,256-275`;
   `content-persistence.md:15-52`). The newly stated `RngStreamId` range is
   internally contradictory, however; see New Finding 3.

### New Findings

1. **Restore and public-replay publication have no live replay-stream
   bootstrap contract.** A `ReplayStreamKey` is mandatory in every
   `WorldBuilder`, may not be derived or replaced, and is said to reach every
   header and tick request (`interfaces.md:208-214,421-423`;
   `content-persistence.md:716-720`). But stream-pair/tombstone reservation,
   sequence-zero append, and durability-before-publication are specified only
   for genesis, with a header hard-coded to tick zero
   (`interfaces.md:281-308`; `content-persistence.md:737-750`). Durable restore
   can instead publish an arbitrary saved frontier and immediately permit its
   next tick (`content-persistence.md:570-604`), while public replay publishes
   its final replayed frontier (`content-persistence.md:802-822`). Neither
   lifecycle includes `ExportingReplayHeader`, reserves the selected stream
   pair/tombstone, defines its first sequence/request range, waits for header
   durability, or defines sink-failure behavior
   (`interfaces.md:1236-1237`). Consequently the first newly confirmed tick
   after restore/replay has no specified well-formed appendable stream: it
   either appends without a header/known sequence or requires Moria to invent
   continuation state.

   Select one concrete contract for each publication route. If each starts a
   new stream, reserve the pair and tombstone before the first sink call,
   define a sequence-zero header anchored to the actual restored/replayed
   starting frontier, and withhold world publication until it is durable. If
   a route continues an existing stream, add the exact consumer-supplied
   durable prefix/next-sequence identity and validation needed to do so without
   inference. In either case, specify rejection ownership, sink failure,
   shutdown, and terminal receipt behavior, and add restore/replay continuation
   fixtures that prove the first later tick is durably ordered after the
   correct header/prefix.

2. **The closed facade cannot represent several promised admission, pending,
   and capacity outcomes.** All generic admission rejections expose only
   `AdmissionError { code: AdmissionCode, retryability }`
   (`interfaces.md:389-479,1893-1910`). Yet `TECH-019` promises rejection
   classifications named `BeforeNext`, `AfterNext`, `WorldNotReady`, and
   `InvalidBatch`, none of which is an `AdmissionCode` variant under those
   names (`interfaces.md:869-871`). `TECH-022` promises `InterestTooLarge`,
   which exists in no closed public error enum (`interfaces.md:1306-1313`).
   More materially, `TECH-023` promises
   `Pending(ReadinessReason)` although `ReceiptState::Pending` can carry only
   `OperationPhase`, and `TECH-024` promises structured
   `ResultCapacityExceeded { required, supported }` although
   `QueryUnavailable` is merely `OperationError` with a fieldless
   `ErrorCode::ResultCapacityExceeded`
   (`interfaces.md:1067-1080,1405-1409,1420-1425,1782-1785,1946-1960`).
   This also fails the approved requirements to report the lifecycle/budget
   condition preventing a query and to reject oversized queries with supported
   bounds (`design-document.md:443-459,917-923`).

   Define one coherent closed error/progress representation reachable through
   the normative `TECH-070` signatures. Align every tick rejection name with
   that representation; give interest/query bound failures their exact bounded
   required/supported context; and make cold/materializing/failed query
   availability observable while pending or terminal as promised. Update the
   public-type index and external compile/use/failure-matrix tests so these
   named outcomes are constructed and pattern-matched rather than asserted
   only in prose.

3. **`RngStreamId` has two incompatible normative domains.** `TECH-005`
   initially defines it, and `TECH-016` encodes it, as a nonzero `u32` scoped
   to one participant (`architecture.md:14-21,595-605`). The new constructor
   contract instead rejects every value above `0x7fff_ffff`
   (`architecture.md:78-86`). Unlike participant and input-source IDs, RNG
   stream IDs do not enter the shared high-bit ordering namespace, and no
   high-bit reservation is otherwise specified. Thus
   `RngStreamId::try_from_raw(0x8000_0000)` is simultaneously valid and
   invalid under the same stable contract.

   Choose and state one domain. Either permit every nonzero `u32`, or explicitly
   make the high-bit reservation part of the RNG wire/descriptor contract and
   apply it consistently. The external constructor fixture must name
   `RngStreamId` specifically and assert the selected boundary.

### Questions For Coder

None. These are ordinary architecture/API closure choices, not protected human
authority questions.

### Answers To Coder

No coder questions were pending. I verified all four claimed corrections
against the actual TDD; three are fully resolved, and the fourth now has a
usable constructor surface but introduced the narrower RNG-range conflict
recorded above.

### Notes To Coder

Preserve the fixed replay-failure record, the separate retired-stream pool, and
the scoped GPU validation semantics. Mechanical traceability still passes:
70 unique `TECH-001` through `TECH-070` headings, exactly one `Implements:`
line per contract, 351 normative contract/requirement pairs, all 43 approved
requirements, no `REQ-042`, and exact parity with `traceability.md`.
`git diff --check` passed before this required ledger append; the worktree was
otherwise clean.

### Approval

Not approved.

## Coder Turn — 2026-07-29T18:50:25Z

Mode: continue

Responding to: 2026-07-29T18:37:43Z

### Responses To Findings

1. Addressed. `TECH-017`, `TECH-021`, `TECH-046`, and `TECH-047` now cover
   replay-stream bootstrap for every world-publication route. Durable restore
   reserves the builder's fresh stream pair/tombstone, writes a sequence-zero
   header anchored to the exact checkpoint store/key/manifest digest and
   restored frontier, and publishes only after matching durability. Public
   replay reserves its fresh stream at admission and, after semantic
   verification, durably copies the exact verified source header and records
   as sequences zero through `N` before the final bundle swap. The new
   `ReplayStreamPosition` on genesis/restore/replay ready results exposes the
   durable prefix and next sequence, so the first later tick is unambiguously
   ordered. Cancellation, sink failure, wrong completion, duplicate pairs,
   tombstone pressure, rejection ownership, and pre/post-invocation cleanup
   are explicit.
2. Addressed. `TECH-019` now uses the actual closed `AdmissionCode` names and
   supplies tick/batch context. `TECH-022` represents `InterestTooLarge` with
   exact required/supported brick counts. `TECH-023` and `TECH-024` define
   `OperationProgress`, `ProgressBlocker`, `QueryReadinessReason`,
   `QueryCapacity`, and the closed `QueryUnavailable` variants, so
   cold/materializing/revision/resource blockers and terminal availability or
   result-capacity failures are callable and pattern-matchable. `TECH-027`
   defines the matching `AdmissionContext` invariants and all formerly
   prose-only admission codes.
3. Addressed. `TECH-005` now consistently gives `RngStreamId` the complete
   nonzero `u32` domain. It rejects only zero and accepts
   `0x8000_0000..=u32::MAX` because the ID is participant-local and never
   enters the shared high-bit ordering namespace. `TECH-060` names the exact
   zero, high-bit, and maximum constructor fixtures.

### TDD Changes Made

- Revised `architecture.md` to make the `RngStreamId` constructor and wire
  domain consistent.
- Revised `interfaces.md` for shared genesis/restore/replay stream
  reservations, replay-bootstrap receipt phases, durable stream positions,
  typed admission context, exact interest/query capacity records, and
  structured query progress/failure.
- Revised `content-persistence.md` with genesis/checkpoint replay anchors,
  restore header durability, checkpoint-anchor replay restoration, verified
  public-prefix copying, sequence/digest invariants, and failure cleanup.
- Revised `validation.md` with external pattern-match tests for every new
  progress/error shape, explicit RNG boundaries, nonzero-tick restore
  continuation, checkpoint-anchor replay, and first-post-publication append
  ordering.

### Questions For Auditor

None.

### Notes To Auditor

No stable `TECH` ID or `Implements:` link changed. Mechanical checks find 70
unique `TECH-001` through `TECH-070` contracts, exactly one `Implements:` line
per contract, 351 normative requirement/contract pairs, all 43 approved
requirements, no `REQ-042`, and exact parity with `traceability.md`.
`git diff --check` passes.

## Auditor Turn — 2026-07-29T18:58:18Z

Mode: continue

Responding to: 2026-07-29T18:50:25Z

### Prior Findings Status

1. **Restore and public-replay stream bootstrap — resolved.** `TECH-017`
   reserves the frozen builder's fresh stream pair and eventual tombstone for
   genesis, restore, and public replay (`interfaces.md:281-321`).
   `TECH-046` withholds a restored world until its checkpoint-anchored
   sequence-zero record is durable and exposes the resulting next sequence
   (`content-persistence.md:566-620`). `TECH-047` verifies a public replay
   privately, then durably copies the exact source header and records as
   sequences zero through `N` before its one bundle publication
   (`content-persistence.md:850-886`). The receipt matrix and validation
   fixtures cover cancellation, failure, reservation cleanup, and the first
   subsequent tick (`interfaces.md:1249-1264`; `validation.md:172-181,408-423`).
2. **Closed admission, query-progress, and query-capacity outcomes —
   resolved.** Tick admission now uses the actual `AdmissionCode` variants and
   contextual tick/batch data (`interfaces.md:876-881`); interest and query
   capacity failures retain exact required/supported records
   (`interfaces.md:1326-1346,1425-1470,1492-1501`); and pending query blockers
   plus terminal availability/capacity outcomes are representable through
   `OperationProgress`, `QueryReadinessReason`, and `QueryUnavailable`
   (`interfaces.md:1077-1209,1433-1451,2069-2107`). The external facade tests
   pattern-match these shapes (`validation.md:86-97`).
3. **`RngStreamId` domain — resolved.** `TECH-005` now consistently makes the
   ID participant-local and accepts every nonzero `u32`, explicitly including
   the high-bit range (`architecture.md:14-21,78-90`). `TECH-016` retains the
   same nonzero wire domain (`architecture.md:598-623`), and `TECH-060` tests
   zero, the high-bit boundary, and `u32::MAX` (`validation.md:133-141`).

### New Findings

1. **The TDD consumes tick zero as genesis, contradicting the approved
   pre-tick genesis boundary.** Approved `REQ-027` says verified construction
   publishes genesis and then “Tick zero begins after that boundary”
   (`design-document.md:271-288`); `REQ-008` likewise requires the world to
   become ready *for* tick zero (`design-document.md:397-420`). The TDD instead
   says genesis itself “publishes tick zero,” returns a tick-zero
   `GenesisReady.frontier`, fixes `next_tick` to one, and labels the genesis
   replay header as tick zero (`interfaces.md:257-265,523-527`;
   `content-persistence.md:765-781`). Under `TECH-011`'s
   `current_tick + 1` eligibility rule, the first post-genesis batch is
   therefore tick one, not the authority-mandated tick zero
   (`architecture.md:373-410`). This is not a harmless display convention:
   tick numbers enter batch eligibility, outcomes, hashes, replay, observation,
   persistence, and rollback. Represent the pre-tick genesis frontier
   concretely without spending tick zero, make the first canonical batch tick
   zero, and propagate the selected representation through genesis receipts,
   replay-header ranges, restore/replay continuity, encoding, and validation.

2. **The closed facade still promises terminal/error outcomes its public error
   type cannot represent.** `TECH-070` says synchronous telemetry returns
   `WorldUnknown`, `WorldClosed`, or `TelemetryBusy`
   (`interfaces.md:692-697`), but `TelemetryError` is only an alias of
   `OperationError`, and the closed `ErrorCode` has neither `WorldUnknown` nor
   `TelemetryBusy` (`interfaces.md:1822-1873,2080-2090`). Separately,
   `TickReceipt::poll` fails with `TickOperationError` (also
   `OperationError`), while the normative lifecycle table promises
   `Failed(FailedNoAdvance)` even though no such public type or error variant is
   defined (`interfaces.md:1164-1167,1223-1227,1251-1255,2080-2082`).
   Select one closed representation for each promised result: either add the
   missing typed outcomes and their scope/retry/committed invariants, or replace
   every promise with the exact existing `OperationError` encoding. For the
   tick-global no-advance path, specify how the failed tick and structured
   cause (including participant/provider identity where relevant) are retained.
   Add external compile/use cases that pattern-match telemetry unknown/busy and
   each tick-global failure route through the actual public type.

3. **Mechanical public-contract closure still has dangling or ambiguous names.**
   `ReplayCompleted.sequence_digest` is a public field with no defined byte
   domain, algorithm, covered range, or relationship to the newly precise
   `ReplayStreamPosition.durable_prefix_digest`
   (`content-persistence.md:690-702,837-843`). An implementation cannot know
   what value that field promises; define it exactly and test it, or remove it
   as redundant. In addition, persistence and recovery bounds refer to
   nonexistent `max_log_ticks`, `max_log_bytes`, and
   `recovery_replay_cap` fields rather than the declared
   `rollback.log_ticks`, `rollback.log_bytes`, and
   `rollback.recovery_replay_ticks`
   (`content-persistence.md:499-505,970-996`;
   `gpu-runtime.md:411-424`; `interfaces.md:174-189`), and presentation
   overflow names undefined `PresentationState::Failed` instead of the
   selected public status/lifecycle type
   (`collision-presentation.md:508-510`; `interfaces.md:1668-1681`).
   Normalize these references to the exact declared fields/types and extend
   the mechanical public-name/budget-field check so stale aliases cannot pass
   the implementation-completeness gate.

### Questions For Coder

None. The genesis numbering is fixed by approved authority, and the remaining
items are ordinary API and naming closure rather than protected human choices.

### Answers To Coder

No coder questions were pending. I verified all three claimed corrections
against the actual TDD; each prior finding is resolved as described above.

### Notes To Coder

Preserve the new stream-bootstrap state machines, exact query blocker/capacity
records, and full nonzero RNG-stream domain. The mechanical provenance audit
passes independently: the seven normative TDD files contain 70 unique
`TECH-001` through `TECH-070` headings, exactly one `Implements:` line per
contract, 351 unique requirement/contract pairs, all 43 approved requirements,
no invented `REQ-042`, and exact pair parity with `traceability.md`. Those
syntactic links remain semantically applicable; the defects above are
authority-numbering and callable-contract closure. `git diff --check` passed
and the worktree was clean before this required ledger append.

### Approval

Not approved.

## Coder Turn — 2026-07-29T19:14:16Z

Mode: continue

Responding to: 2026-07-29T18:58:18Z

### Responses To Findings

1. Addressed. `TECH-010`, `TECH-011`, `TECH-014`, `TECH-017`, `TECH-019`,
   `TECH-046`, `TECH-047`, and `TECH-070` now represent genesis with the
   explicit `FrontierPosition::Genesis`, distinct from
   `FrontierPosition::Confirmed(Tick::from_raw(0))`. Genesis publishes the
   pre-tick frontier and returns `next_tick == 0`; batch zero is the first
   canonical transition and produces the first confirmed frontier. Replay
   headers now use `ReplayAppendRange::Header { starting, next_tick }` rather
   than fabricating a tick-zero range. Genesis, checkpoint restore, and public
   replay validate the same checked next-tick function, and the tests cover
   genesis/confirmed-zero encoding, hash, admission, rollback, and append
   ordering.
2. Addressed. `TECH-027` now defines the closed `TelemetryError` variants
   `WorldUnknown`, `WorldClosed`, and `TelemetryBusy` with fixed world,
   retryability, and no-effect semantics. Tick receipts now fail with the
   concrete `FailedNoAdvance` record, which retains attempted tick, exact
   source frontier, structured canonical/participant/provider/device/shutdown/
   internal cause, and a matching `OperationError` whose committed effect is
   `None`. `TECH-060` requires external pattern matches for every telemetry
   variant and every tick-global failure route, including participant/provider
   identity and device generation.
3. Addressed. The redundant undefined `ReplayCompleted.sequence_digest` field
   is removed; `ReplayStreamPosition.durable_prefix_digest` is the sole prefix
   digest and retains its exact BLAKE3 tuple domain, with independent golden
   tests. Normative references now use `rollback.log_ticks`,
   `rollback.log_bytes`, `rollback.recovery_replay_ticks`, and
   `PresentationStatus::Failed`. `TECH-060` adds a mechanical public-name and
   nested-budget-field check with the prior stale names retained as negative
   fixtures.

### TDD Changes Made

- Revised `architecture.md` and `overview.md` for the pre-tick genesis source,
  tick-zero transition, frontier-position hashing, and rollback accounting.
- Revised `interfaces.md` with `FrontierPosition`, checked next-tick accessors,
  genesis/confirmed result invariants, `FailedNoAdvance`,
  `TickNoAdvanceCause`, concrete `TelemetryError`, and exact public-type index
  ownership.
- Revised `content-persistence.md` with header-versus-tick replay append
  ranges, genesis/restore/replay continuity, one precise prefix digest, and
  exact rollback budget-field references.
- Revised `gpu-runtime.md` and `collision-presentation.md` to use the callable
  tick-global failure and selected budget/presentation names.
- Revised `validation.md` with genesis-to-tick-zero, telemetry, no-advance,
  replay-prefix, and mechanical stale-name/field fixtures.

### Questions For Auditor

None.

### Notes To Auditor

The accepted stream-bootstrap, query blocker/capacity, and full nonzero
`RngStreamId` contracts are preserved. No stable `TECH` ID or `Implements:`
link changed. Mechanical checks find 70 unique `TECH-001` through `TECH-070`
contracts, exactly one `Implements:` line per contract, 351 normative
requirement/contract pairs, all 43 approved requirements, no `REQ-042`, and
exact parity with `traceability.md`. `git diff --check` passes.

## Auditor Turn — 2026-07-29T19:21:53Z

Mode: continue

Responding to: 2026-07-29T19:14:16Z

### Prior Findings Status

1. **Pre-tick genesis and tick-zero authority — resolved.** The world root now
   commits the closed `Genesis | Confirmed(Tick)` frontier and `TECH-011`
   admits tick zero from genesis without spending that tick
   (`architecture.md:366-383`). The facade gives genesis its own encoding,
   checked next-tick rule, and ready-result invariant
   (`interfaces.md:512-542,692-702`); replay headers likewise distinguish the
   pre-tick anchor from the first tick record
   (`content-persistence.md:769-810,867-877`). The new headless fixture checks
   both sides of that boundary (`validation.md:107-116`).
2. **Concrete telemetry and tick-global failure APIs — resolved.**
   `TickReceipt` now returns the closed `FailedNoAdvance`, which retains the
   attempted tick, source frontier, structured provider/participant/device
   cause, and `OperationError` (`interfaces.md:1966-2003`). Its invariants bind
   scope, committed effect, cause, and retryability
   (`interfaces.md:2185-2194`). `TelemetryError` now has the three promised
   pattern-matchable variants and explicit retry semantics
   (`interfaces.md:1992-2003,2196-2200`), with external contract tests required
   in `validation.md:98-105`.
3. **Replay digest and stale public-contract names — resolved.**
   `ReplayStreamPosition.durable_prefix_digest` is now the sole exposed prefix
   digest and has an exact ordered tuple domain
   (`content-persistence.md:867-874`). Normative persistence, recovery, and
   presentation references use the declared nested budget/status names, and
   the mechanical closure test retains the former aliases only as negative
   fixtures (`validation.md:202-208`).

### New Findings

1. **Rollback correction has no implementable replay-log or durable-stream
   branch contract.** Approved `REQ-035` expressly allows corrected inputs to
   produce a new tick/hash sequence, and `REQ-032`/`REQ-038` require genesis
   plus the complete ordered log to reproduce the current hash sequence
   (`design-document.md:803-827,1094-1123`). `TECH-047` defines only a linear
   append stream whose post-header records are ordinary single-tick records
   (`content-persistence.md:769-822`). `TECH-048` privately replays replacement
   batches and swaps the corrected frontier, but never says how the superseded
   in-memory suffix is replaced, how the already-durable append-only suffix is
   invalidated or branched, which corrected records receive stream sequences,
   or when those records become durable
   (`content-persistence.md:969-996`). Its statement that participant events
   appear “in the replay record” names no record variant or append operation.
   As written, appending replacement ticks after the old suffix creates
   duplicate/noncontiguous tick numbers that public replay rejects, while
   retaining the old suffix makes checkpoint reconstruction of a corrected
   reconstructible participant use the wrong bytes. Select a concrete bounded
   branch/reset representation and publication policy across `TECH-047`–
   `TECH-049`: define in-memory suffix replacement, durable stream records and
   sequence/digest evolution, reconstruction/checkpoint ownership,
   cancellation and append-failure outcomes, and whether corrected-frontier
   publication waits for durability or follows a precisely reported
   post-publication terminal policy. Add a cold public-replay and checkpoint
   restore fixture in which corrected inputs change hashes and only the
   exported corrected history reproduces the live frontier.

2. **The explicit genesis frontier has not propagated through all canonical
   fact and participant wire contracts.** `TECH-070` forbids a sentinel tick
   for genesis and explicitly permits querying the genesis frontier
   (`interfaces.md:692-701`), yet the “complete” canonical `CollisionFact`
   still contains only `tick: Tick` (`collision-presentation.md:157-180`).
   A collision query against genesis must therefore either mislabel its source
   as confirmed tick zero or leave a canonical wire field without a value.
   Participant state has the same mismatch: `TECH-016` still binds every token
   to a bare tick even though genesis preparation creates the source token
   (`architecture.md:542-549`), while the public metadata has already selected
   `FrontierSummary` (`interfaces.md:2342-2349`). The GPU ABI compounds the
   ambiguity by requiring tick words for all operations, including `Genesis`,
   without defining their operation-specific value or validity
   (`collision-presentation.md:251-275,375-385`). Replace source-position
   fields with the closed frontier representation where they identify state,
   or explicitly separate attempted tick from source frontier and define every
   operation's encoding. Extend the genesis fixture to query collision truth
   and exercise CPU/GPU genesis participant token metadata, proving
   byte-level distinction from `Confirmed(Tick(0))`.

### Questions For Coder

None. Both findings are ordinary consistency and persistence engineering
within already approved authority; neither requires a protected human
decision.

### Answers To Coder

No coder questions were pending. I verified all three claimed corrections
against the committed TDD; each prior finding is resolved as recorded above.

### Notes To Coder

Preserve the new pre-tick frontier, concrete failure APIs, replay bootstrap,
query blocker/capacity records, and full nonzero `RngStreamId` domain. The
mechanical provenance audit still passes independently: 70 unique
`TECH-001`–`TECH-070` contracts, one `Implements:` line per contract, 351
unique contract/requirement pairs, all 43 approved requirements, no
`REQ-042`, and exact parity with `traceability.md`. `git diff --check` passed
and the worktree was clean before this required ledger append.

### Approval

Not approved.

## Coder Turn — 2026-07-29T19:45:24Z

Mode: continue

Responding to: 2026-07-29T19:21:53Z

### Responses To Findings

1. Addressed. `TECH-047` now defines a third physical replay record,
   `CorrectionBranch`, whose one atomic append contains the exact framed
   corrected tick records and the target, superseded, corrected, previous
   active-history, and corrected active-history commitments. Folding the
   append-only physical stream removes the superseded semantic suffix without
   deleting its diagnostic bytes. `TECH-048` admits only a complete replacement
   through the existing present, requires the current stream to be durable and
   idle, bounds the one branch record before private work, and waits for branch
   durability before one mandatory `FrontierBundle`/rollback-deque/active-log/
   stream-position publication. Cancellation ends at sink invocation; earlier
   failures preserve the original live world, while append failure preserves
   that frontier but terminally fails the world under the existing exact replay
   provider-failure record. `TECH-049` makes the in-memory log the active
   projection with physical sequence/subrecord locators. Checkpoints extract
   only active exact tick frames, including frames embedded in a branch, and
   bind them to the physical stream and active-history digest. Public replay
   folds and copies physical branch records exactly. Validation now cold-replays
   a corrected stream and restores a post-correction checkpoint after discarding
   all live state.
2. Addressed. `TECH-052::CollisionFact` and the `TECH-053` collider header/view
   now carry `FrontierPosition`, so Genesis facts are byte-distinct from
   confirmed tick zero. `TECH-016` binds every participant token to the closed
   frontier position. The CPU tick request now separates attempted `tick` from
   `source_frontier` and validates their checked next-tick relation. The GPU
   participant ABI is now a fixed 224-byte wire with independently tagged
   optional source frontier, destination frontier, and attempted tick, exact
   offsets/zero rules, and a closed per-operation validity table for Genesis,
   Tick, snapshot restore, reconstruction, and snapshot export. The genesis
   fixture now queries collision and inspects CPU/GPU token metadata on both
   sides of the Genesis/Confirmed(0) boundary.

### TDD Changes Made

- Revised `architecture.md` for frontier-bound participant tokens and the
  correction durability precondition on publication.
- Revised `content-persistence.md` for atomic correction branches, semantic
  replay folding, physical sequence/digest evolution, active-log splicing,
  checkpoint record ownership, cancellation/failure behavior, and cold replay.
- Revised `interfaces.md` for correction phases/results/observations,
  source-frontier CPU participant inputs, and collider views.
- Revised `collision-presentation.md` for frontier-bearing facts/artifacts and
  the complete 224-byte operation-specific GPU wire.
- Revised `gpu-runtime.md`, `overview.md`, and `validation.md` for the atomic
  correction publication bridge, ownership summary, Genesis byte fixtures, and
  corrected replay/checkpoint scenarios.

### Questions For Auditor

None.

### Notes To Auditor

The correction design deliberately retains one consumer-selected live stream:
an append-only branch record changes only its semantic projection, so it
requires neither deletion nor an inferred replacement stream identity. A
single-record branch is rejected before private work if it cannot fit the
configured correction, log, sink-record, or in-flight byte bounds. No stable
`TECH` ID or `Implements:` link changed. Mechanical checks still find 70
unique `TECH-001` through `TECH-070` contracts with one heading each;
`git diff --check` passes.

## Auditor Turn — 2026-07-29T19:52:46Z

Mode: continue

Responding to: 2026-07-29T19:45:24Z

### Prior Findings Status

1. **Rollback correction replay-log and durable-stream branch contract —
   resolved.** `TECH-047` now defines an atomic physical
   `CorrectionBranch`, its exact fold into one active semantic history, the
   physical-prefix and active-history digest domains, and cold public replay
   behavior (`content-persistence.md:774-833,921-987`). `TECH-048` admits a
   complete bounded suffix only while the existing stream is durable and
   idle, waits for branch durability, then coordinates the corrected
   frontier, rollback deque, active log, stream position, receipt, and
   observation in one publication (`content-persistence.md:1039-1117`).
   Checkpoint extraction retains exact corrected subrecords and their physical
   locators (`content-persistence.md:394-409,486-531`), and validation now
   covers pending/failing branch export, cold replay, and post-correction
   checkpoint restore (`validation.md:433-489`).
2. **Genesis frontier propagation through collision and participant wires —
   resolved.** `CollisionFact` and collider artifacts now carry the closed
   source frontier and explicitly distinguish Genesis from confirmed tick zero
   (`collision-presentation.md:165-225`). Participant tokens and CPU tick
   requests bind source and destination positions
   (`architecture.md:548-556`; `interfaces.md:2324-2331,2501-2511`). The
   224-byte GPU ABI specifies exact position tags, offsets, zero rules, and a
   validity row for every operation (`collision-presentation.md:385-422`),
   with byte-level Genesis/confirmed-zero fixtures in
   `validation.md:107-122`.

### New Findings

1. **Correction-branch append failure has contradictory committed-effect
   semantics.** `TECH-048` says this failure leaves the original bundle and
   active log installed and never advances the correction
   (`content-persistence.md:1127-1136`); the lifecycle matrix likewise says
   only `Ready(CorrectionCommitted)` publishes a corrected frontier
   (`interfaces.md:1299`), and the common rule says only successful
   correction publication can change authority (`interfaces.md:1305-1311`).
   But the shared `TECH-047` provider-failure record used for both an ordinary
   post-confirmation tick append and a prepublication correction branch sets
   `OperationError.committed` to
   `Frontier(the_last_trustworthy_frontier)`
   (`content-persistence.md:875-888`). Because `CorrectionError` is exactly
   `OperationError` (`interfaces.md:2165-2173`), a failed correction receipt
   can therefore report a committed frontier even though its own operation
   committed none. Separate the two cases: define the exact
   `CorrectionReceipt` error and `WorldLifecycleFact.failure` values for branch
   failure, with `CommittedEffect::None` on the correction operation (or
   otherwise revise the closed committed-effect semantics coherently), while
   preserving the already-confirmed tick case. Add external pattern-match
   assertions that the receipt, correction observation, lifecycle fact, and
   last readable frontier all agree.
2. **`CorrectionRequest.expected_hashes` has no closed cardinality or index
   contract.** The public request exposes independent bounded
   `replacement_batches` and `expected_hashes` vectors
   (`content-persistence.md:1016-1023`), but the only rule is “when supplied,
   [they] are checked at each tick” (`content-persistence.md:1060-1067`).
   This does not decide whether an empty vector disables comparison, whether a
   partial vector is legal, or which hash corresponds to which replacement
   tick; nor does it name the rejection outcome for a mismatched count. Define
   one exact rule—most simply, empty or exactly one hash per replacement batch
   in the same contiguous tick order—plus checked admission, ownership return,
   and fixtures for empty, exact, short, and excess vectors. This closes the
   callable correction API required by `REQ-035` without adding behavior.

### Questions For Coder

None. Both findings are ordinary public-contract closure within the selected
correction design.

### Answers To Coder

No coder questions were pending. I verified both claimed corrections against
the committed TDD; both prior findings are resolved as recorded above.

### Notes To Coder

Preserve the append-only physical branch with its active semantic projection,
checkpoint subrecord locators, durability-before-publication rule, and the
closed Genesis/confirmed frontier encodings. The mechanical provenance audit
passes independently: 70 unique `TECH-001`–`TECH-070` contracts, one
`Implements:` line per contract, 351 unique contract/requirement pairs, all 43
approved requirements, no `REQ-042`, and exact parity with
`traceability.md`. `git diff --check` passed and the worktree was clean before
this required ledger append.

### Approval

Not approved.

## Coder Turn — 2026-07-29T20:02:25Z

Mode: continue

Responding to: 2026-07-29T19:52:46Z

### Responses To Findings

1. Addressed. `TECH-047` now distinguishes an already-confirmed tick append
   failure from a prepublication correction-branch append failure.
   `TECH-048` and `TECH-027` define a closed `CorrectionError` containing the
   unchanged original frontier, an `OperationError` whose committed effect is
   always `None`, and the exact optional `ReplayExportFailure`. Branch failure
   emits a matching correction observation and world-lifecycle fact, while
   `WorldLifecycleFact.frontier` explicitly records the last readable
   trustworthy frontier. The ordinary tick case retains
   `CommittedEffect::Frontier(the_confirmed_tick_frontier)`.
2. Addressed. `CorrectionRequest.expected_hashes` is now legal only when empty
   or exactly equal in length to `replacement_batches`. Empty disables the
   optional comparison; otherwise index `i` maps to replacement batch `i` and
   contiguous tick `target + 1 + i`, and compares against that candidate
   frontier's root hash before private advance. Nonempty short or excess
   vectors return the unchanged owned request with the typed
   `CorrectionHashCountMismatch` and exact batch/hash counts before any pin,
   permit, participant callback, or sink invocation.

### TDD Changes Made

- Revised `content-persistence.md` with operation-relative committed-effect
  semantics, exact correction failure records, hash cardinality/index
  mapping, mismatch behavior, and lossless rejection ownership.
- Revised `interfaces.md` with the specialized `CorrectionError`,
  `WorldLifecycleFact.frontier`, and closed correction hash-count admission
  code/context.
- Revised `validation.md` with external pattern matches across correction
  receipt/observation/lifecycle/readable frontier, the distinct ordinary-tick
  export failure, and empty/exact/short/excess expected-hash fixtures.

### Questions For Auditor

None.

### Notes To Auditor

The append-only physical correction branch, active-history fold,
durability-before-publication rule, and Genesis/confirmed encodings are
unchanged. No stable `TECH` ID or `Implements:` link changed. Mechanical checks
find 70 unique contracts, one `Implements:` line each, and exact 351-pair
parity with `traceability.md`; `git diff --check` passes.

## Auditor Turn — 2026-07-29T20:06:17Z

Mode: approved

Responding to: 2026-07-29T20:02:25Z

### Prior Findings Status

1. **Correction-branch append failure committed-effect semantics —
   resolved.** `TECH-047` now distinguishes an ordinary post-confirmation tick
   append failure, whose causal tick already committed, from a prepublication
   correction-branch failure (`content-persistence.md:883-905`).
   `CorrectionError` retains the unchanged original frontier and requires
   `CommittedEffect::None`; branch failure additionally carries the exact
   `ReplayExportFailure` and matching provider-scoped operation error
   (`interfaces.md:1984-1988,2227-2241`;
   `content-persistence.md:1163-1188`). `WorldLifecycleFact.frontier`
   independently records the last trustworthy live frontier
   (`interfaces.md:1737-1752`). The external-style validation fixture
   pattern-matches the correction receipt, correction observation, lifecycle
   fact, readable frontier, and the distinct ordinary-tick case
   (`validation.md:433-453`).
2. **`CorrectionRequest.expected_hashes` cardinality and index contract —
   resolved.** `TECH-048` permits exactly an empty vector or one hash per
   replacement batch, maps index `i` to the contiguous corrected tick and its
   candidate root, returns the complete owned request on a typed
   `CorrectionHashCountMismatch`, and performs rejection before pins, permits,
   participant work, or sink invocation
   (`content-persistence.md:1030-1085`). `TECH-027` defines the closed
   admission code/context pair (`interfaces.md:2087-2093,2103-2121,
   2207-2216`), and validation covers empty, exact, divergent, short, and
   excess vectors with ownership preservation (`validation.md:462-473`).

### New Findings

None. The complete TDD is implementation-ready against the approved product
design and committed decisions.

### Questions For Coder

None.

### Answers To Coder

No coder questions were pending. I verified both claimed corrections against
the committed TDD; both prior findings are resolved as recorded above.

### Notes To Coder

Approved GDD coverage and approved implementation completeness are separate
conclusions, and both pass this review. The TDD gives every material approved
capability a concrete public or internal representation, bounded ownership and
failure behavior, coherent authority/publication rules, portable GPU and
participant contracts, persistence/replay semantics, and executable validation
obligations without adding product scope or a privileged consumer path.

The independent provenance audit found 70 unique contracts spanning
`TECH-001` through `TECH-070`, exactly one `Implements:` line per contract, 43
approved requirements covered, no invented `REQ-042`, 351 unique
contract/requirement pairs, and exact parity with `traceability.md`.
`git diff --check` passed and the worktree was clean before this required
ledger append.

### Approval

Approved.

## Coder Turn — determinism addendum revision

Mode: continue

Responding to: human-review feedback preserved verbatim in
`docs/tdd/decisions.md` under “Human review entry — determinism addendum”

### Responses To Findings

1. **Cross-machine tier — removed.** The former `TECH-063` cross-backend
   qualification matrix is deleted and its ID retired. `TECH-039`,
   `TECH-040`, `TECH-069`, the facade, persistence, overview, and repository
   commands no longer require qualification manifests, per-driver renewal,
   cross-vendor comparison, or backend-family CI gates. Native adapter support
   remains a portability/runtime capability, not a cross-machine determinism
   claim.
2. **Replay-grade core — preserved.** Canonical tick publication,
   event-sourced commands, rollback roots, incremental hierarchical hashing,
   durable replay, correction branches, and divergence artifacts retain their
   stable contracts. `ExecutionPolicy::ReplayGrade` now states the exact
   same-machine/same-genesis/same-`TickBatch`/bit-identical-hash guarantee.
3. **Kernel contamination audit — preserved and made explicit.** `TECH-035`,
   `TECH-061`, and `TECH-062` retain the complete source/dataflow audit and
   same-machine scheduling perturbations for atomics, races, padding,
   allocation identity, and compaction order. No audit obligation was removed
   with the cross-vendor matrix.
4. **Fixed point — retained with revised verification rationale.** Canonical
   code remains float-free. Golden math/replay bytes use specified integer
   operations and are no longer tied to a qualification row or GPU-driver
   identity.
5. **Parameterized placement — addressed.** `TECH-007` defines the per-world
   frozen fractional split, cell extent, and simulation-unit identity,
   includes them in the configuration fingerprint/genesis/checkpoint/replay,
   and rejects mismatches. `TECH-016`/`TECH-029` add bounded participant-owned
   representation-contract declarations for every non-placement physical
   quantity.
6. **Canonical fixed-point math component — addressed.** New `TECH-071`
   defines `moria-fixed-v1`, its generic CPU/WGSL implementations, 64-bit
   intermediate multiplication, exact rounding/division/square root, canonical
   CORDIC trig, typed boundary, and arbitrary-precision differential evidence.

### Concrete TDD Evidence

- The normative set has 70 active stable contracts: surviving
  `TECH-001`–`TECH-062`, `TECH-064`–`TECH-070`, plus new `TECH-071`.
  `TECH-063` is explicitly retired and not reused.
- Every active contract has exactly one `Implements:` line. `REQ-039` remains
  visible in traceability only as superseded by the supplied human amendment;
  no active contract claims to implement its removed cross-machine gate.
- `TECH-059` tests all placement fractional splits against an
  arbitrary-precision reference and regenerates the CORDIC constants.
  `TECH-061` audits every canonical WGSL entry point and helper.
  `TECH-062` performs at least eight perturbed byte-identical replays on the
  same physical machine.
- `TECH-004`'s exact local commands include shader validation and same-machine
  replay verification; no cross-vendor matrix command remains.
- The public facade freezes `WorldGenesisConfig.placement`, exposes distinct
  raw integer types only, and binds participant representation contracts
  before genesis.

### Questions For Reviewer

None.

### Notes To Reviewer

The older ledger entries are preserved as the historical coding-agent/review-
agent conversation. Their cross-GPU references describe the then-current TDD
and are not represented as current human authority. The new human feedback is
preserved separately and verbatim in `decisions.md`.

## Auditor Turn — 2026-07-29T20:42:04Z

Mode: continue

Responding to: none

### Prior Findings Status

No prior auditor finding remained open: the last timestamped auditor turn
approved the then-current TDD. This turn reviews the later determinism
addendum revision. The coder's removal of the cross-machine tier, retirement
of `TECH-063`, retention of replay-grade tick/hash/rollback behavior, retention
of the kernel-contamination audit, parameterized placement format, and bounded
participant representation declarations are verified in the current files.

### New Findings

1. **`TECH-071` does not yet specify one implementable, byte-unique CORDIC
   transition.** The contract fixes the table-generation formulas, iteration
   count, intermediate width, and final rounding, but says only “quadrant
   reduction” and refers to an iteration order “displayed ... in the generated
   source,” which is not part of this TDD
   (`architecture.md:396-410`). It does not select the exact conversion from
   `TurnQ32` into the internal angle word, quadrant/boundary ownership, initial
   CORDIC state, positive/negative/zero residual branch rule, per-iteration
   update equations, or final quadrant remap. The public
   `QuatQ14::try_from_axis_turn` additionally says only “half-angle” without
   fixing modular halving or the zero-axis outcome
   (`architecture.md:330-350`). CPU and WGSL implementations can therefore
   follow the stated constants and 32 iterations yet disagree at zero
   residuals, exact quadrant boundaries, and axis-angle edge inputs. Tests over
   those boundaries cannot supply missing normative semantics
   (`validation.md:22-29`). Put the complete integer recurrence and all edge
   rules in `TECH-071` (generated code may implement it but may not define it),
   name the exact `CanonicalFailure` for a zero/unrepresentable axis, and add
   retained CPU/WGSL golden vectors for every quadrant boundary, zero
   residual, maximum turn word, and axis-angle zero/half/full-turn cases.
2. **The durable replay header's new “execution identity” is undefined and can
   contradict the diagnostic-only adapter boundary.** `TECH-047` lists
   `execution identity` as an exact common field of the sequence-zero replay
   header without naming its wire type, fields, encoding, or replay comparison
   rule (`content-persistence.md:801-813`). The only apparent candidate is
   `ExecutionSummary`, which contains `adapter_context`
   (`gpu-runtime.md:530-534`). But adapter/driver identity is explicitly
   excluded from the configuration fingerprint
   (`architecture.md:465-475`), is only run context in replay evidence
   (`validation.md:331-340`), and a driver update does not create a new replay
   identity (`gpu-runtime.md:562-568`). Encoding `adapter_context` as identity
   would make otherwise identical replay headers and prefix digests
   machine/driver-specific; ignoring it leaves public replay and
   `ContractMismatch` behavior undefined. Replace “execution identity” with a
   closed header record and exact compatibility rules. Keep authority status
   and the configuration fingerprint if required, but either exclude adapter/
   driver context from durable replay identity or explicitly mark and encode
   it as non-comparison diagnostic data without allowing it to reject replay.
   Add a fixture proving that changed adapter context alone neither changes the
   canonical replay identity nor causes a pre-transition mismatch, while a
   changed placement/arithmetic/configuration fingerprint does.

### Questions For Coder

None. Both findings are ordinary deterministic-math and replay-format closure
under the supplied human addendum.

### Answers To Coder

No coder question was pending. The later human amendment is accepted as the
authority for retiring the cross-machine requirement; I did not re-raise
`REQ-039` merely because the older approved design text remains in history.

### Notes To Coder

Preserve the addendum's scope decision: no cross-vendor matrix, per-driver
qualification, multiplayer claim, or reused `TECH-063`. Also preserve the
same-machine replay promise, contamination audit, parameterized world
placement, participant-owned representation contracts, and distinct canonical
numeric types.

The independent provenance audit found 70 active unique headings:
`TECH-001`–`TECH-062`, `TECH-064`–`TECH-071`; one `Implements:` line per
active contract; 346 unique active contract/requirement pairs with exact
parity to `traceability.md`; all 43 design requirements represented there,
with `REQ-039` explicitly superseded by the later human amendment and no
invented `REQ-042`. `git diff --check` passed and the worktree was clean before
this required ledger append.

### Approval

Not approved.

## Coder Turn — 2026-07-29T20:50:26Z

Mode: continue

Responding to: 2026-07-29T20:42:04Z

### Responses To Findings

1. Addressed. `TECH-071` now defines the complete `TurnQ32` to signed
   2^62-turn conversion, nearest-quadrant midpoint ownership, initial
   gain-inverse state, increasing 32-step simultaneous recurrence,
   floor-shift semantics, the `zi >= 0` zero-residual branch, final
   ties-to-even Q1.30 reduction, and the four exact quadrant remaps. Generated
   Rust/WGSL implements this TDD recurrence rather than defining it.
   `TECH-007` now fixes unsigned floor half-angle construction, including odd
   words and the zero/full-turn alias; exact Q1.30 axis normalization and raw
   quaternion construction; and `ZeroAxis` versus `UnrepresentableAxis`
   failures. `TECH-059` retains CPU/WGSL per-iteration and final golden bytes
   at every quadrant center/reduction midpoint, adjacent words, the maximum
   turn word, and the requested axis-angle edges.
2. Addressed. `TECH-047` replaces the undefined execution identity with the
   exact 33-byte `ReplayIdentityV1 { authority_status,
   configuration_fingerprint }`, fixes its tags and digest domain, and defines
   pre-transition equality and `ContractMismatch` behavior for genesis,
   restore, and public replay. Adapter/device/backend/driver/process/worker
   context and candidate fault plans are absent from header and replay digest
   bytes and cannot reject replay. `ExecutionSummary.adapter_context` remains
   noncomparison telemetry/divergence context. `TECH-066` proves changed
   adapter context preserves sequence-zero/identity/prefix bytes while changed
   placement, arithmetic/table data, or another configuration input fails
   before callbacks, device work, transitions, or sink invocation.

### TDD Changes Made

- Revised `architecture.md` with the normative CORDIC recurrence, exact
  boundary ownership, axis normalization, half-angle construction, and typed
  axis failures.
- Revised `interfaces.md` with appended stable `CanonicalFailure` variants and
  diagnostic-only adapter-context wording.
- Revised `content-persistence.md` with the closed durable replay identity,
  exact wire encoding/digest, compatibility checks, and context exclusions.
- Revised `gpu-runtime.md` to point replay identity changes to `TECH-047`
  without tying them to driver updates.
- Revised `validation.md` with retained CPU/WGSL CORDIC/axis-angle goldens and
  replay-identity compatibility/mismatch fixtures.

### Questions For Auditor

None.

### Notes To Auditor

The later human addendum remains intact: there is no cross-machine,
cross-vendor, per-driver qualification, multiplayer, or reused `TECH-063`
contract. Mechanical checks still find 70 active unique headings
(`TECH-001`–`TECH-062`, `TECH-064`–`TECH-071`), exactly one `Implements:` line
per contract, 346 unique contract/requirement pairs with exact traceability
parity, and all 43 design requirements represented with `REQ-039` explicitly
superseded. `git diff --check` passes.

## Auditor Turn — 2026-07-29T20:55:33Z

Mode: approved

Responding to: 2026-07-29T20:50:26Z

### Prior Findings Status

1. **`TECH-071` deterministic CORDIC and axis-angle closure — resolved.**
   `TECH-071` now fixes the `TurnQ32` widening and signed `2^62`-per-turn
   domain, nearest-quadrant reduction and midpoint ownership, initial state,
   checked simultaneous 32-step recurrence, zero-residual branch, floor-shift
   rule, final ties-to-even reduction, and exact quadrant remap
   (`architecture.md:411-472`). It also defines exact Q1.30 axis
   normalization, zero and checked-intermediate failures, and the exhaustive
   representability boundary (`architecture.md:474-489`). `TECH-007` fixes
   unsigned half-angle construction, validates the axis even for an identity
   angle, names `ZeroAxis` and `UnrepresentableAxis`, and specifies raw
   quaternion construction without premature narrowing
   (`architecture.md:344-365`). Those variants are appended without changing
   existing canonical-failure tags (`interfaces.md:2158-2179,2220-2222`).
   Retained CPU/WGSL vectors cover centers, midpoints and adjacent words,
   per-iteration state, the maximum turn word, zero/basis/extreme axes, odd
   half-angle truncation, and zero/half/full-turn cases
   (`validation.md:23-48`).
2. **Closed replay identity and diagnostic adapter boundary — resolved.**
   `TECH-047` replaces the undefined execution identity with the exact
   33-byte `ReplayIdentityV1`, closed authority-status tags, the frozen
   configuration fingerprint, and its BLAKE3 domain
   (`content-persistence.md:801-836`). Restore and public replay compare the
   identity and already enumerated canonical header fields before transition,
   callbacks, device submission, or destination-sink invocation, with
   `ContractMismatch` on any mismatch
   (`content-persistence.md:838-845`). Adapter, device, backend, driver,
   process, worker, and fault-plan context are expressly absent from header and
   replay digest bytes and cannot reject replay
   (`content-persistence.md:847-855`); `ExecutionSummary.adapter_context`
   remains diagnostic-only (`gpu-runtime.md:530-534`). `TECH-066` now proves
   byte identity under adapter-context-only changes and pre-transition failure
   under configuration or status changes (`validation.md:530-539`).

### New Findings

None. The complete current TDD is implementation-ready under the later human
determinism addendum.

### Questions For Coder

None.

### Answers To Coder

No coder question was pending. I verified both claimed corrections in the
committed TDD; both prior findings are resolved as recorded above.

### Notes To Coder

The determinism addendum is applied without weakening replay-grade canonical
ticks, rollback, hashing, replay artifacts, or the kernel-contamination audit,
and without retaining the removed cross-machine tier. The current design keeps
adapter/driver identity diagnostic, leaves `TECH-063` retired, and introduces
no multiplayer or privileged consumer path.

The independent provenance check found 70 active unique headings:
`TECH-001`–`TECH-062`, `TECH-064`–`TECH-071`; exactly one `Implements:` line
per contract; 346 unique contract/requirement pairs with exact parity to
`traceability.md`; no duplicate pair or invented requirement; and every
approved requirement represented except `REQ-039`, whose removal is explicitly
authorized and recorded. `git diff --check` passed and the worktree was clean
before this required ledger append.

### Approval

Approved.
