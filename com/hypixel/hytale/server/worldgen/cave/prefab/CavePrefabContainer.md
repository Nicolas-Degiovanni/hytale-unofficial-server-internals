---
description: Architectural reference for CavePrefabContainer
---

# CavePrefabContainer

**Package:** com.hypixel.hytale.server.worldgen.cave.prefab
**Type:** Utility / Data Model

## Definition
```java
// Signature
public class CavePrefabContainer {
```

## Architecture & Concepts

The CavePrefabContainer is a high-level, immutable data structure that serves as a blueprint for procedural cave decoration. It does not perform any generation itself; rather, it holds a collection of rules and weighted prefab choices that are consumed by active world generation systems.

This class embodies a data-driven design pattern. It decouples the *logic* of world generation (the Java code that places blocks) from the *design* (the specific prefabs to place, where, and how often). An instance of CavePrefabContainer typically represents a single, deserialized world generation asset file, such as a JSON definition.

The core architectural concept is a hierarchy of configuration:

1.  **Container:** The top-level object, holding an array of entries.
2.  **CavePrefabEntry:** A single rule that associates a collection of potential prefabs with a specific configuration for their placement.
3.  **CavePrefabConfig:** The heart of the rule, containing a battery of conditions and suppliers. These include biome masks, height constraints, noise density checks, and placement strategies.
4.  **WorldGenPrefabSupplier:** A reference to a specific prefab structure that can be placed in the world.

This structure allows designers to create complex, layered cave features by defining multiple entries, each targeting different environmental conditions.

## Lifecycle & Ownership

-   **Creation:** Instances are not created directly in procedural code. They are instantiated by a deserialization service (e.g., GSON) during the server's asset loading phase. The constructor is populated with data parsed from world generation configuration files.
-   **Scope:** An instance of CavePrefabContainer is long-lived. It persists for the entire lifetime of the server's world generation context, acting as a read-only configuration source.
-   **Destruction:** The object is eligible for garbage collection when the server shuts down or a world is unloaded, at which point the entire world generation context is discarded.

## Internal State & Concurrency

-   **State:** Deeply immutable. The `entries` array and all fields within the nested `CavePrefabEntry` and `CavePrefabConfig` classes are declared as `final`. This design guarantees that the configuration, once loaded, cannot be altered at runtime.
-   **Thread Safety:** This class is inherently thread-safe. Its immutability allows it to be safely read by multiple concurrent chunk generation threads without any need for locks, mutexes, or other synchronization primitives. This is a critical performance characteristic for a massively parallel system like world generation.

## API Surface

The primary interaction is through iterating its entries and evaluating the conditions within each entry's config.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getEntries() | CavePrefabEntry[] | O(1) | Returns the master list of all prefab placement rules. |
| CavePrefabEntry.getPrefab(random) | WorldGenPrefabSupplier | O(log N) | Selects a prefab from the weighted map based on a random value. |
| CavePrefabConfig.getHeight(...) | int | O(N) | Calculates the final Y-coordinate for placement, including displacement. |
| CavePrefabConfig.isMatchingBiome(biome) | boolean | O(1) | Evaluates if the rule applies to the given biome. |
| CavePrefabConfig.isMatchingNoiseDensity(...) | boolean | O(1) | Evaluates a procedural noise condition at a given coordinate. |
| CavePrefabConfig.getIterations(random) | int | O(1) | Determines how many placement attempts should be made for this entry. |

## Integration Patterns

### Standard Usage

The canonical use case involves a cave generation task iterating through the container's entries for a given world location. For each entry, the generator evaluates all conditions. If they pass, it selects and places a prefab.

```java
// Pseudo-code for a cave generator task
CavePrefabContainer container = worldGenContext.getCavePrefabs("deep_caves");
Random random = new Random(chunkSeed);

for (CavePrefabContainer.CavePrefabEntry entry : container.getEntries()) {
    CavePrefabConfig config = entry.getConfig();

    // Evaluate all placement conditions
    if (config.isMatchingBiome(currentBiome) && config.isMatchingNoiseDensity(seed, x, z)) {
        int y = config.getHeight(seed, x, z, caveNode);

        if (y != -1 && config.isMatchingHeight(seed, x, y, z, random)) {
            // Conditions met, select a prefab and place it
            WorldGenPrefabSupplier prefab = entry.getPrefab(random.nextDouble());
            if (prefab != null) {
                world.placePrefab(prefab, x, y, z, config.getRotation(random));
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never construct a CavePrefabContainer manually using `new`. These objects are intended to be created exclusively from asset file deserialization. Manual creation bypasses the data-driven design and hardcodes world generation content.
-   **State Mutation:** Do not attempt to modify the returned `entries` array or any of its contents. The system relies on the immutability of this data for thread safety. Any modification via reflection or other means will lead to severe and difficult-to-debug concurrency issues.

## Data Pipeline

This class acts as a configuration source within the broader world generation pipeline. It does not process data itself but provides the rules that guide other systems.

> Flow:
> WorldGen Asset File (JSON) -> Asset Deserializer -> **CavePrefabContainer** -> Cave Generation Task -> Prefab Placement in World Chunk

