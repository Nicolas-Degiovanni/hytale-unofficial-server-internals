---
description: Architectural reference for the PropsSource interface, a core contract in the procedural world generation system.
---

# PropsSource

**Package:** com.hypixel.hytale.builtin.hytalegenerator.biome
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface PropsSource {
```

## Architecture & Concepts
The PropsSource interface is a fundamental contract within the procedural world generation engine. It establishes a standardized data provider pattern for any entity capable of defining decorative elements, known as *props* (e.g., trees, rocks, bushes), and their placement rules within a given area.

Its primary architectural role is to decouple the high-level **World Generator** from the low-level **Biome Definitions**. The generator does not need to know the concrete type of a biome; it only needs to interact with an object that fulfills the PropsSource contract. This allows the system to query any biome, sub-biome, or special zone for its prop data in a uniform way, enabling a highly extensible and data-driven generation pipeline.

Implementations of this interface, typically Biome configuration objects, serve as immutable data carriers, loaded once during the engine's asset initialization phase.

### Lifecycle & Ownership
As an interface, PropsSource itself has no lifecycle. The following pertains to its concrete implementations, such as a specific Biome class.

-   **Creation:** Implementations are almost exclusively instantiated and populated by a configuration loader or asset manager during the server or client bootstrap sequence. They are typically deserialized from world generation asset files (e.g., JSON or HOCON definitions).
-   **Scope:** An instantiated PropsSource implementation is a long-lived, effectively singleton object within the context of a loaded world. It persists for the entire session, serving as a read-only source of truth for biome data.
-   **Destruction:** Instances are garbage collected when the world asset context is unloaded, typically upon server shutdown or when a client disconnects from a world.

## Internal State & Concurrency
-   **State:** Implementations of PropsSource are expected to be **deeply immutable**. The lists returned by its methods should be unmodifiable collections that are populated once at creation and never change. This guarantees deterministic and reproducible world generation.
-   **Thread Safety:** Implementations **must be thread-safe for reads**. The world generator operates across multiple worker threads to build world chunks in parallel. These threads will concurrently access the same PropsSource instance to retrieve prop data for their assigned chunks. Failure to ensure thread safety will result in severe generation artifacts and potential crashes. The immutability requirement inherently provides this thread safety.

## API Surface
The public contract is minimal, focusing solely on data retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPropFields() | List<PropField> | O(1) | Returns the definitive list of all prop types that can spawn in this context. Expected to be a fast, non-blocking call. |
| getAllPropDistributions() | List<Assignments> | O(1) | Returns the complete set of rules, weights, and conditions governing how the props from getPropFields are placed. |

## Integration Patterns

### Standard Usage
The World Generator or a subordinate Biome Placement service retrieves the appropriate PropsSource for a given block column and uses its data to drive the prop population stage.

```java
// Assume 'biomeRegistry' holds all loaded biome definitions
// and 'currentBiomeId' is determined for a specific world coordinate.

PropsSource currentBiome = biomeRegistry.getBiome(currentBiomeId);

List<PropField> availableProps = currentBiome.getPropFields();
List<Assignments> placementRules = currentBiome.getAllPropDistributions();

// The generator's prop populator now uses this data
// to place props within a chunk.
propPopulator.populateChunk(chunk, availableProps, placementRules);
```

### Anti-Patterns (Do NOT do this)
-   **Mutable Implementations:** Never design an implementation where the lists returned by the getters can be modified after instantiation. This breaks the deterministic nature of the world generator and is a critical architectural violation.
-   **Expensive Getters:** The getter methods must not perform on-the-fly calculations, disk I/O, or any other expensive operations. They are expected to be simple accessors to pre-loaded, in-memory data.
-   **Dynamic Instantiation:** Do not use `new` to create implementations of PropsSource during the game loop or generation process. All definitions should be loaded and registered once at startup.

## Data Pipeline
PropsSource acts as a conduit for static configuration data to be fed into the dynamic, procedural generation algorithms.

> Flow:
> Biome Definition File (JSON) -> Asset Deserializer -> **PropsSource Implementation** -> World Generator's Prop Placement Stage -> Final Chunk Voxel & Entity Data

