---
description: Architectural reference for CoverContainer
---

# CoverContainer

**Package:** com.hypixel.hytale.server.worldgen.container
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class CoverContainer {
```

## Architecture & Concepts

The CoverContainer is a passive, immutable data structure that serves as a blueprint for the "cover" layer of procedural world generation. It does not contain any active logic itself; instead, it encapsulates a hierarchical set of rules that a world generation orchestrator consumes to decorate the surface of generated terrain with elements like grass, snow, flowers, or small rocks.

Architecturally, this class sits at the boundary between the asset loading system and the world generation pipeline. It is the in-memory representation of a biome's surface decoration rules, typically deserialized from a configuration file (e.g., JSON).

The system is designed as a hierarchy of conditional rules:
1.  A **CoverContainer** holds an array of one or more rule sets.
2.  Each **CoverContainerEntry** within the array represents a distinct rule set with its own set of conditions, such as biome, height, and the type of block it can be placed upon.
3.  If all conditions for a CoverContainerEntry are met, it provides a weighted map of **CoverContainerEntryPart** objects, which define the final block to be placed.

This structure allows for complex and varied surface generation by layering multiple conditional rules within a single biome definition.

### Lifecycle & Ownership
-   **Creation:** CoverContainer instances are not intended for direct instantiation within game logic. They are created by a higher-level asset management or biome definition system during server bootstrap or when biome assets are loaded. The data is populated by deserializing biome-specific configuration files.
-   **Scope:** An instance of CoverContainer persists for as long as its corresponding biome definition is loaded in memory. In most scenarios, this means it is cached for the entire server session to avoid repeated file I/O and deserialization.
-   **Destruction:** The object is eligible for garbage collection when the server shuts down or if the asset system performs a hot-reload of biome definitions. There is no explicit destruction method.

## Internal State & Concurrency
-   **State:** The CoverContainer and its nested classes are deeply immutable. All internal fields are declared as final, and the constructor is the sole point of state initialization. This design guarantees that a loaded ruleset cannot be altered at runtime.
-   **Thread Safety:** This class is inherently thread-safe due to its immutability. A single CoverContainer instance can be safely read by multiple world generation worker threads simultaneously without any need for locks, synchronization, or defensive copies. This is a critical feature for enabling high-performance, parallelized chunk generation.

## API Surface

The primary API is the structure of the data itself. Consumers interact with the object by retrieving its rule sets and evaluating them.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getEntries() | CoverContainerEntry[] | O(1) | Returns the top-level array of rule sets for this container. |
| CoverContainerEntry.get(Random) | CoverContainerEntryPart | O(N) | Selects a specific block part based on internal weights. N is the number of parts. |
| CoverContainerEntry.getParentCondition() | IBlockFluidCondition | O(1) | Retrieves the condition that must be met by the block underneath. |
| CoverContainerEntry.getMapCondition() | ICoordinateCondition | O(1) | Retrieves the condition based on world X/Z coordinates. |
| CoverContainerEntry.getHeightCondition() | ICoordinateRndCondition | O(1) | Retrieves the condition based on world Y coordinate and a random factor. |
| CoverContainerEntry.getCoverDensity() | double | O(1) | Returns the probability (0.0 to 1.0) that this rule set will be applied. |

## Integration Patterns

### Standard Usage

The intended consumer is a world generation service responsible for populating a chunk. The service retrieves the appropriate CoverContainer for the chunk's biome and iterates through its rules for each relevant surface column.

```java
// Pseudocode for a world generation orchestrator
void applyCoverLayer(Chunk chunk, Biome biome) {
    CoverContainer coverRules = biome.getCoverContainer();
    Random random = new Random(chunk.getSeed());

    for (CoverContainer.CoverContainerEntry rule : coverRules.getEntries()) {
        // Iterate over all surface blocks in the chunk
        for (int x = 0; x < 16; x++) {
            for (int z = 0; z < 16; z++) {
                // Check if the rule's conditions are met for this block
                if (conditionsMet(rule, x, z, random)) {
                    // Probabilistically apply the cover based on density
                    if (random.nextDouble() < rule.getCoverDensity()) {
                        CoverContainer.CoverContainerEntry.CoverContainerEntryPart part = rule.get(random);
                        if (part != null) {
                            placeBlock(x, z, part.getEntry(), part.getOffset());
                        }
                    }
                }
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new CoverContainer()` in procedural generation logic. These objects must only be created from authoritative biome asset files. Manual creation can lead to world generation that is inconsistent with its definition.
-   **State Modification:** Do not attempt to modify the internal state of a CoverContainer or its children using reflection. This violates the immutability contract and will cause severe, unpredictable concurrency issues in the world generation engine.
-   **Ignoring Conditions:** Applying a `CoverContainerEntry` without first evaluating its associated `mapCondition`, `heightCondition`, and `parentCondition` will result in incorrect block placement, such as flowers appearing underwater or grass growing on stone.

## Data Pipeline

The CoverContainer is a key link in the chain that transforms declarative asset data into in-game world geometry.

> Flow:
> Biome Definition Asset (JSON) -> Asset Deserialization Service -> **CoverContainer** (In-Memory Object) -> World Generation Orchestrator -> Final Chunk Block Data

---
---
description: Architectural reference for CoverContainer.CoverContainerEntry
---

# CoverContainer.CoverContainerEntry

**Package:** com.hypixel.hytale.server.worldgen.container
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public static class CoverContainerEntry {
```

## Architecture & Concepts

The CoverContainerEntry represents a single, self-contained "rule set" for placing a specific category of surface cover. It is the primary logical unit within the CoverContainer system, bundling a set of conditions with a potential set of outcomes.

A world generator evaluates a list of these entries for each surface block. The first entry whose conditions are fully met is typically chosen to be processed. This class acts as a conditional gate, ensuring that cover elements are placed only in appropriate contexts—for example, a rule for "snow cover" would contain a `parentCondition` for non-foliage blocks and a `mapCondition` for a cold biome.

The `coverDensity` field is a key architectural element, providing a probabilistic control layer. It decouples the *eligibility* of a block for cover from the *actual placement*, allowing for sparse or patchy distribution of surface elements, which creates a more natural look.

### Lifecycle & Ownership
-   **Creation:** Instantiated exclusively during the creation of its parent CoverContainer, typically via deserialization from a biome asset file.
-   **Scope:** Its lifecycle is strictly bound to its parent CoverContainer. It exists as long as the parent object exists.
-   **Destruction:** Cleaned up by the garbage collector when its parent CoverContainer is destroyed.

## Internal State & Concurrency
-   **State:** Fully immutable. All fields are final and set at construction time.
-   **Thread Safety:** Inherently thread-safe. It can be read by any number of threads without synchronization, which is essential for parallel chunk generation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(Random) | CoverContainerEntryPart | O(N) | Selects a block part from the weighted map. Throws if the map is empty. |
| getParentCondition() | IBlockFluidCondition | O(1) | Gets the predicate for the block *underneath* the target position. |
| getMapCondition() | ICoordinateCondition | O(1) | Gets the predicate for the global X/Z coordinates. |
| getCoverDensity() | double | O(1) | Gets the probability (0.0-1.0) that this rule will be applied if conditions pass. |
| getHeightCondition() | ICoordinateRndCondition | O(1) | Gets the predicate for the Y coordinate, which may include a random factor. |
| isOnWater() | boolean | O(1) | Returns true if this cover can be placed on a water block. |

## Integration Patterns

### Standard Usage

The world generator iterates through the entries provided by a CoverContainer. For each entry, it evaluates the conditions in a specific order, typically starting with the cheapest checks.

```java
// Pseudocode for evaluating a single entry
void evaluateRule(CoverContainer.CoverContainerEntry rule, BlockPos pos, Random random) {
    // 1. Check biome/map conditions (often done at a higher level)
    if (!rule.getMapCondition().test(pos.getX(), pos.getZ())) {
        return;
    }

    // 2. Check the block this cover would sit on
    BlockState parentBlock = world.getBlockState(pos.down());
    if (!rule.getParentCondition().test(parentBlock)) {
        return;
    }

    // 3. Check height conditions
    if (!rule.getHeightCondition().test(pos.getY(), random)) {
        return;
    }

    // 4. All conditions passed, now apply probabilistically
    if (random.nextDouble() < rule.getCoverDensity()) {
        CoverContainer.CoverContainerEntry.CoverContainerEntryPart part = rule.get(random);
        if (part != null) {
            world.setBlockState(pos.up(part.getOffset()), part.getEntry().getBlockState());
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Assuming Outcome:** Calling `get(random)` before verifying that all conditions have passed is wasteful and logically incorrect.
-   **Ignoring Random Instance:** Passing a new `Random()` instance for every call to `get(random)` or `getHeightCondition().test()` can lead to visual artifacts and non-deterministic world generation. A seeded, per-chunk or per-region Random instance must be used.

