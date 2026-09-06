# Moria project boundary

Moria's product is the reusable voxel-world substrate, exposed as a Rust crate
or a small family of tightly scoped Rust crates.

The actual game is a separate downstream consumer and is not part of this
repository. Moria may include a walkable-world executable, but that executable
is only a validation harness. It must consume the substrate through the same
public interfaces available to an external game rather than owning privileged
or game-specific implementation paths.

This is an immediate, concrete reason to use a Cargo workspace boundary between
the reusable substrate and its validation harness. The precise crate split is a
technical-design decision; the consumer boundary is not optional.

Game rules and the future System, LLM, spell, gas, combat, AI, and building
layers are out of scope. Compatibility seams may be designed where the substrate
requirements demand them, but those layers must not be implemented here.
