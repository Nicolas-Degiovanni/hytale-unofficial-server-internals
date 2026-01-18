---
description: Architectural reference for MaskProvider
---

# MaskProvider

**Package:** com.hypixel.hytale.server.worldgen.chunk
**Type:** Transient

## Definition
```java
// Signature
public class MaskProvider {
```

## Architecture & Concepts
The MaskProvider is a high-level, immutable facade over the `FuzzyZoom` system, which is the core engine for procedural zone and biome placement. It serves as a clean, queryable interface for the rest of the world generation pipeline, abstracting away the complexities of the underlying noise and layering algorithms.

Its primary architectural role is to represent a specific, static "mask" of the world at a particular stage of generation. For example, an initial MaskProvider might define the layout of major biomes like forests and deserts. Subsequent operations can then use this base mask to generate a *new*, more detailed MaskProvider that layers unique zones, such as dungeons or villages, on top of the existing layout.

This class is not a service but rather a data-transfer object that carries a snapshot of the world's zone configuration. Its immutability is a critical design feature, enabling safe, parallel processing by multiple chunk generation threads without the need for locks or synchronization.

## Lifecycle & Ownership
-   **Creation:** A MaskProvider is instantiated by a higher-level world generation coordinator, typically during the initialization of a `Zone` or a specific generation pass. It is always created with a fully configured `FuzzyZoom` instance, which encapsulates the rules for that layer of the world.
-   **Scope:** The lifetime of a MaskProvider instance is intentionally short. It is scoped to a single, discrete stage of the world generation process. Once a new MaskProvider is created from it (e.g., via `generateUniqueZones`), the original instance is typically discarded.
-   **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as the generation stage that created it completes and no longer holds a reference. There are no explicit cleanup or disposal methods.

## Internal State & Concurrency
-   **State:** Immutable. The class holds a single `final` reference to a `FuzzyZoom` object. This state is set at construction and cannot be changed for the lifetime of the object. All methods are either pure query functions or factory methods that produce a new MaskProvider instance.
-   **Thread Safety:** This class is inherently thread-safe. Due to its immutable nature, a single MaskProvider instance can be safely shared and read by multiple worker threads simultaneously during parallel chunk generation. This design is fundamental to the performance of the world generator.

## API Surface
The public API focuses on querying spatial data and advancing the generation pipeline.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, y) | int | O(1) | Retrieves the zone identifier for a given world coordinate. This is the primary query method. |
| distance(x, y) | double | O(1) | Calculates the distance from a point to the nearest zone boundary, useful for smooth transitions. |
| inBounds(x, y) | boolean | O(1) | Verifies if a coordinate is within the valid operational area of the underlying zoom algorithm. |
| generateUniqueZoneCandidates(...) | Zone.UniqueCandidate[] | O(N) | Scans the map to identify a list of potential spawn locations for unique, non-repeating zones. |
| generateUniqueZones(...) | MaskProvider | O(N) | Consumes a list of candidates, finalizes their positions, and returns a **new** MaskProvider representing the next stage of the world map. |

## Integration Patterns

### Standard Usage
The MaskProvider is designed to be used in a chained or layered fashion. A base provider is created, queried, and then used as a factory for a more refined provider.

```java
// 1. Obtain the base MaskProvider for the current generation stage.
MaskProvider baseProvider = zoneGenerator.getInitialMaskProvider();

// 2. Use the provider to determine the zone for a specific block.
int zoneId = baseProvider.get(worldSeed, blockX, blockZ);

// 3. Generate candidates for a special feature.
Zone.UniqueCandidate[] candidates = baseProvider.generateUniqueZoneCandidates(...);

// 4. Create a new, refined provider by placing the unique zones.
MaskProvider refinedProvider = baseProvider.generateUniqueZones(worldSeed, candidates, random, placedZones);

// 5. Subsequent generation logic MUST use the new refinedProvider.
int newZoneId = refinedProvider.get(worldSeed, blockX, blockZ);
```

### Anti-Patterns (Do NOT do this)
-   **State Modification:** Do not attempt to retrieve the internal `FuzzyZoom` object and modify its state. The system relies on the immutability of the MaskProvider for thread safety and predictable generation.
-   **Stale References:** Do not continue to use an old MaskProvider after a new one has been generated from it via `generateUniqueZones`. The old provider does not contain the updated world information and will lead to inconsistent or incorrect generation results.
-   **Direct Instantiation:** Avoid creating a MaskProvider with a manually constructed `FuzzyZoom`. The `FuzzyZoom` instances are typically built by a complex, configured chain within the world generator and should be obtained from the appropriate coordinator.

## Data Pipeline
The MaskProvider acts as a queryable snapshot within the larger world generation data flow.

> Flow:
> World Seed & Config -> `FuzzyZoom` Chain -> **MaskProvider (Base)** -> Chunk Generator Queries -> `generateUniqueZones` -> **MaskProvider (Refined)** -> Chunk Generator Queries -> Zone ID -> Block Placement Logic

