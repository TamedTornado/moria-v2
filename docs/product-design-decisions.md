# Moria product design decisions

This record preserves human decisions made while reviewing the product design.
The approved `docs/product-vision.md` remains authoritative for product
identity, boundary, requirements, constraints, and non-goals. The standalone
design document states each current outcome in full; this record preserves the
review history rather than replacing design substance.

## Resolved decisions

### D-001. Multi-target matter mutation completion

**Status:** Resolved by human product-design review.

**Question considered:** When one bounded matter-mutation command targets
multiple cells, may it report partial application, or must it commit as one
atomic public operation?

**Decision:** The command is atomic. All targeted matter changes become
committed together at one revision, or none of them commit. A consumer that
wants independently successful effects submits separate commands.

**Design consequences:** Admission and internal work may be staged, but
queries, collision, observations, persistence, and presentation never observe
a partially committed command. A failed admitted matter mutation has no
committed effect. Validation must force a multi-cell command to fail after
admission and confirm that no targeted cell, mutation revision, or intermediate
observation escaped.

**Boundary retained:** This selects consumer-visible semantics only. Staging,
coordination, rollback, and other mechanisms that provide atomicity remain
technical-design concerns.

### D-002. Mandatory canonical tick authority

**Status:** Resolved by human product-design review.

**Decision:** Deterministic tick authority is mandatory for every Moria world.
World construction and verified content installation establish a canonical
genesis state. After genesis, every operation capable of changing canonical
state belongs to exactly one numbered tick and enters through one versioned,
canonically ordered tick batch. There is no nondeterministic convenience
mutation path.

**Boundary retained:** This is simulation authority, not a game loop or
networking protocol. Consumers decide how ticks are paced and how input reaches
Moria. Technical design selects the encoding and transition mechanisms.

### D-003. Canonical simulation representation

**Status:** Resolved by human product-design review.

**Decision:** Canonical transitions use integer or exactly specified
fixed-point semantics. Canonical results may not depend on floating-point
variation, races, iteration order, worker identity, completion timing, or
unspecified arithmetic behavior. Canonical placement supports arbitrary
three-dimensional rigid orientation through a representation closed under
canonical composition and inverse. Its precision must keep one orientation
quantization step below one cell of displacement at the maximum supported
volume radius.

**Boundary retained:** Widths, scales, encodings, overflow rules, coordinate
mechanisms, and the orientation representation are technical-design choices
that must satisfy these product semantics.

### D-004. Local replay and cross-GPU determinism

**Status:** Resolved by human product-design review.

**Decision:** Identical canonical genesis bytes and ordered tick-batch bytes
produce identical canonical results and per-tick hashes within one qualified
backend and across every GPU vendor, driver, and backend tuple Moria claims as
qualified. Local replay determinism and cross-GPU determinism are separate
named invariants. Qualification is fail-closed and driver-version specific.

**Boundary retained:** Rendered pixels and derived presentation need not be
deterministic. Unqualified backends may not claim deterministic authority even
if they render or pass local replay tests.

### D-005. Bounded rollback and performance qualification

**Status:** Resolved by human product-design review.

**Decision:** Moria retains a configurable rollback window with a universal
minimum capacity of 20 confirmed ticks. A retained snapshot shares unchanged
world state and restore installs a retained canonical frontier without copying
or traversing the whole voxel world. Performance qualification measures the
complete restore-and-replay correction on a declared adversarial
constraint-chain workload. Completing 20 ticks within the declared
simulation-frame interval earns that performance tier; slower correct hardware
reports its lower measured capability rather than failing deterministic
correctness or claiming the tier.

**Boundary retained:** Workload population, dirty rate, frame interval,
hardware classes, retained-state mechanisms, and measured-curve format are
technical-design parameters. The constraint chain is a qualification workload,
not Moria gameplay.

### D-006. Incremental hashes and public replay

**Status:** Resolved by human product-design review.

**Decision:** Every confirmed tick has a canonical simulation hash covering
all state capable of changing a future canonical result, including coordinated
participant commitments. Hash maintenance is incremental with work following
changed state rather than whole-world traversal. Canonical genesis plus the
complete ordered input log reproduces the canonical hash sequence. Replay and
an earliest-divergence artifact are public debugging and validation
capabilities, not test-only helpers.

**Boundary retained:** Hash algorithm, hierarchy representation, replay storage
format, and diagnostic transport remain technical-design choices. Their
domains and contract versions must be explicit.

### D-007. Derived-cache freedom and deterministic collision

**Status:** Resolved by human product-design review.

**Decision:** Presentation and other derived caches are outside the canonical
determinism boundary and cannot influence simulation. Collision facts used by
deterministic behavior are a pure, tick-synchronous derivation of canonical
matter and placement. A conforming GPU participant may consume canonical
occupancy/collision inputs without mandatory CPU readback; a CPU participant
may consume a canonical collider artifact bound to its source hash.

**Boundary retained:** Moria does not prescribe a physics engine, collision
algorithm, renderer, or cache technique. Readiness timing never chooses whether
canonical collision exists.

### D-008. Coordinated participant rollback

**Status:** Resolved by human product-design review.

**Decision:** Every external behavior participant capable of affecting
canonical state registers one explicit rollback strategy:
`PerTickSnapshot` or `ReconstructibleFromCanonicalStateAndLog`. It contributes
to the coordinated canonical commitment and restores at the same frontier.
There is no default strategy and divergence cannot be accepted silently.

**Boundary retained:** Participants continue to own their vocabulary, state,
algorithms, and effects. Moria coordinates identity, scheduling, commitments,
bounded restoration, and failure without acquiring physics, damage, or
gameplay policy.

### D-009. Canonical simulation-domain lifecycle

**Status:** Resolved by human product-design review.

**Decision:** Simulation-domain activation and deactivation are canonical,
tick-stamped state, distinct from local render, inspection, or materialization
interest. Activation binds exact content identity and fails closed when that
content is absent or mismatched. Overlapping consumer-defined activity regions
resolve to one deterministic union.

**Boundary retained:** Consumers define what activity and any coarse simulation
mean. Moria transports and coordinates that state without defining gameplay.
Technical design may initially require session-scale simulation residency.

## Open human questions

None.

---

## Human review entry

### Verbatim feedback

```text
Multi-target mutation completion: must a bounded mutation affecting multiple cells commit atomically, or may the contract support explicitly reported partial application? This design requires no unreported partial success but does not choose between those public semantics.

Atomic commits seems like the better idea.

Also, a somewhat vague guidance to keep in mind: The GPU residency requirements are about performance. Our physics and other "behaviour" engines will primarily be on the GPU as well, when that is feasible. This engine will be correct first, and then relentlessly optimised.
```

### Decision and clarification

The human selected atomic public completion for a bounded multi-target matter
mutation. One command commits all targeted changes together at one revision or
commits none; explicitly reported partial application is not a supported
outcome.

GPU residency is a performance requirement, not an end in itself. Correctness
and the public truth contract take priority, followed by continuous measured
optimization. The generic behavior-extension boundary must accommodate
physics and other external behavior engines running primarily on the GPU when
feasible, without making those engines part of Moria or granting them ownership
of Moria's voxel storage.

### Unresolved question

None at product-design altitude. The mechanisms and measured thresholds for an
efficient GPU-oriented extension path remain technical-design and validation
choices.

---

## Deterministic-simulation decisions

### Recorded human direction

The human requires Moria to serve as authoritative voxel-world state inside
deterministic simulations with rollback, replay, and cross-machine desync
detection. This direction explicitly preserves GPU-resident authority and
external ownership of physics, damage, generation, networking, and gameplay
policy.

### Decision and clarification

Decisions D-002 through D-009 preserve the settled product calls. The design
must integrate them throughout mutation, lifecycle, collision, persistence,
behavior-extension, failure, validation, and performance experience. Their
technical parameters remain for the technical design to select and prove; they
are not open product-boundary questions.

### Unresolved question

None at product-design altitude.
