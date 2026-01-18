---
description: Architectural reference for PrefabPasteUtil
---

# PrefabPasteUtil

**Package:** com.hypixel.hytale.server.worldgen.prefab
**Type:** Utility

## Definition
```java
// Signature
public class PrefabPasteUtil {
```

## Architecture & Concepts

The PrefabPasteUtil is a stateless, static utility class that serves as the core engine for procedural structure placement, or "pasting," within Hytale's world generation system. It acts as the interpreter that translates an abstract prefab definition into concrete block, entity, and fluid data within a specific chunk context.

Its primary architectural role is to bridge the high-level `WorldGenPrefabSupplier`, which provides prefab data on demand, and the low-level `ChunkGeneratorExecution`, which represents the mutable state of a chunk being generated.

The system is designed around a powerful recursive model. A single call to `generate` can trigger a cascade of subsequent calls if the source prefab contains instructions to place other, "child" prefabs. This allows for the creation of highly complex and varied structures from a set of smaller, reusable components. To prevent stack overflows and runaway generation, this recursion is explicitly limited by `MAX_RECURSION_DEPTH`.

All state required for a paste operation is encapsulated within the `PrefabPasteBuffer` object. This design choice makes PrefabPasteUtil itself completely stateless, ensuring that its methods are predictable and free of side effects beyond the explicit modification of the provided buffer and execution context.

Key features handled by this utility include:
*   **Recursive Prefab Placement:** Placing prefabs that in turn place other prefabs.
*   **Heightmap Fitting:** Adjusting the vertical placement of a prefab to conform to the existing world terrain.
*   **Conditional Placement:** Using a `BlockMaskCondition` to determine if a block from a prefab should overwrite an existing block in the world.
*   **Entity Spawning:** Placing and configuring entities defined within the prefab data.
*   **Deterministic Randomness:** Using world and specific seeds to ensure that procedural choices (like selecting a prefab from a weighted list) are repeatable for a given world.

## Lifecycle & Ownership

-   **Creation:** As a static utility class, PrefabPasteUtil is never instantiated. Its methods are invoked directly on the class.
-   **Scope:** The methods of this class have a transient scope. They are invoked by the world generation pipeline during the construction of a chunk and exist only for the duration of the method call stack.
-   **Destruction:** Not applicable. The class is never destroyed. However, the associated `PrefabPasteBuffer` object, which holds the operational state, is designed to be reset and reused to minimize object allocation overhead during the highly repetitive process of chunk generation.

## Internal State & Concurrency

-   **State:** PrefabPasteUtil is **stateless**. All operational state, including world coordinates, chunk coordinates, random seeds, recursion depth, and placement conditions, is externally managed within the `PrefabPasteBuffer` instance passed into each `generate` call. The `PrefabPasteBuffer` is highly **mutable** and its state is modified extensively during the recursive pasting process.

-   **Thread Safety:** The class itself is thread-safe due to its stateless nature. However, the operations it performs are **not thread-safe** in practice. A single paste operation, initiated by a call to `generate`, must be confined to a single thread. The `ChunkGeneratorExecution` and `PrefabPasteBuffer` objects are not designed for concurrent access. World generation systems must ensure that each worker thread operates on its own distinct instances of these context objects to prevent race conditions and world data corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generate(buffer, rotation, supplier, x, y, z, cx, cz) | static void | O(N\*M) | The primary entry point. Initiates a recursive prefab paste operation. Complexity is dependent on the number of elements N in the prefab and the maximum recursion depth M. |
| PrefabPasteBuffer | static class | - | A mutable context object that holds all state for a single, complete paste operation. It is passed through the entire recursive call stack. |

## Integration Patterns

### Standard Usage

The utility is designed to be called from a higher-level world generation controller, such as a chunk generator. The controller is responsible for managing a pool of `PrefabPasteBuffer` objects, configuring one for each paste operation, and invoking the static `generate` method.

```java
// Obtain a reusable buffer, perhaps from an object pool
PrefabPasteUtil.PrefabPasteBuffer buffer = getBufferFromPool();

// Configure the buffer for a specific paste operation
buffer.setSeed(worldSeed, chunkSpecificSeed);
buffer.execution = myChunkGeneratorExecution;
buffer.priority = GenerationPriorities.STRUCTURES;
// ... other buffer configurations

// Obtain the prefab data supplier
WorldGenPrefabSupplier supplier = prefabLoader.get("structures/village/house_small");

// Execute the paste operation at a specific world location
PrefabPasteUtil.generate(buffer, PrefabRotation.NONE, supplier, worldX, worldY, worldZ, chunkX, chunkZ);

// Return the buffer to the pool for reuse
buffer.reset();
returnBufferToPool(buffer);
```

### Anti-Patterns (Do NOT do this)

-   **Buffer State Leakage:** Reusing a `PrefabPasteBuffer` instance for a new operation without first calling `reset()` is a critical error. This will cause state from the previous operation (e.g., coordinates, rotation, recursion depth) to leak into the new one, resulting in unpredictable and incorrect structure placement.
-   **Concurrent Execution:** Sharing a single `PrefabPasteBuffer` or `ChunkGeneratorExecution` instance across multiple threads is forbidden. This will lead to severe race conditions, data corruption, and unstable server behavior. Each world generation thread must operate on its own isolated set of context objects.
-   **Ignoring Recursion Depth:** Designing prefabs that form deep or infinite recursive loops can lead to performance degradation or stack overflow errors, even with the `MAX_RECURSION_DEPTH` safeguard. Prefab dependency graphs should be carefully managed.

## Data Pipeline

PrefabPasteUtil is a processor stage within the larger world generation data pipeline. It consumes abstract prefab data and produces concrete modifications to a chunk's data arrays.

> Flow:
> World Generator -> Obtains `WorldGenPrefabSupplier` -> Configures `PrefabPasteBuffer` -> **PrefabPasteUtil.generate()** -> Modifies `ChunkGeneratorExecution` (Blocks, Fluids, Entities) -> Final Chunk Data

