---
description: Architectural reference for SpawningContext
---

# SpawningContext

**Package:** com.hypixel.hytale.server.spawning
**Type:** Transient

## Definition
```java
// Signature
public class SpawningContext {
```

## Architecture & Concepts

The SpawningContext is a stateful, short-lived object that encapsulates the entire environment and state required to evaluate a single potential spawn location within the world. It serves as a computational engine for the server's entity spawning system, abstracting the complex process of identifying viable spawn points within a vertical block column.

Its primary architectural role is to act as a bridge between high-level spawning logic (which decides *what* to spawn and generally *where*) and low-level world data and physics systems. Instead of scattering world lookups and geometric calculations throughout the spawning code, this class centralizes the analysis of a specific (X, Z) column.

The core concept is the **Spawn Span**. A Spawn Span is a contiguous vertical range of non-solid blocks (air or fluid) that is tall enough to accommodate a given entity model. The SpawningContext's main function is to scan a world column, identify all such spans, and provide methods to test specific spawn placements (e.g., on the ground, in water) within those spans. It is effectively a state machine, configured sequentially with a spawnable entity, a world chunk, and a target column, before yielding a final, validated spawn position.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly via its public constructor, typically by a higher-level spawner service at the beginning of a spawn evaluation cycle for a given area.
-   **Scope:** The lifecycle of a SpawningContext instance is intended to be brief, covering the evaluation of one or more spawn locations within a single logical operation. The presence of `release` and `releaseFull` methods strongly implies that instances are designed to be reused or pooled to reduce object allocation overhead.
-   **Destruction:** The caller is responsible for explicitly calling `release()` or `releaseFull()` when the context is no longer needed. Failure to do so will result in memory leaks by retaining references to heavy objects like World and WorldChunk.

## Internal State & Concurrency
-   **State:** The SpawningContext is highly **mutable**. Its public fields and internal state are continuously modified by its methods during the evaluation process. It caches references to the world, the target chunk, the entity model, and a calculated list of potential SpawnSpans. This state is built up progressively through a series of method calls.
-   **Thread Safety:** This class is **not thread-safe**. It is designed for synchronous, single-threaded access. Its mutable state and lack of any internal locking mechanisms make it inherently unsafe for concurrent use. A single instance must not be shared across threads without external synchronization, which would defeat its purpose as a high-performance, transient utility.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setSpawnable(ISpawnableWithModel) | boolean | O(Asset) | Configures the context with the entity to spawn. This is a prerequisite for most other operations as it defines the required physical dimensions. |
| setChunk(WorldChunk, int) | void | O(1) | Primes the context with the world chunk where spawning will be attempted. |
| setColumn(x, z, ...) | boolean | O(H) | Scans the vertical column at the given coordinates to find and store all valid SpawnSpans. H is the height of the scan range. |
| selectRandomSpawnSpan() | boolean | O(1) | Selects one of the previously computed spans and populates detailed state like ground level, water level, and air height. |
| canSpawn() | SpawnTestResult | O(Collision) | Performs a final validation using the CollisionModule to check for intersecting blocks or entities at the calculated spawn position. |
| isOnSolidGround() | boolean | O(1) | A placement strategy method. Attempts to find a valid Y-coordinate on a solid surface within the selected span. |
| isInWater(float minDepth) | boolean | O(1) | A placement strategy method. Attempts to find a valid Y-coordinate within water of a minimum depth. |
| release() / releaseFull() | void | O(1) | Clears internal state, releasing references to heavy objects like WorldChunk. Must be called after use to prevent memory leaks. |

## Integration Patterns

### Standard Usage
The SpawningContext is used in a strict, sequential manner. The typical workflow involves configuring the context, finding valid spans, selecting a placement strategy, and finally validating the result.

```java
// 1. Create or acquire a context instance
SpawningContext context = new SpawningContext();

// 2. Configure with the entity to be spawned
if (!context.setSpawnable(someSpawnable)) {
    // Model could not be loaded, abort
    return;
}

// 3. Set the world location
context.setChunk(worldChunk, environmentIndex);
if (!context.setColumn(x, z, yHint, yRange)) {
    // No valid spans found in this column, abort
    context.release();
    return;
}

// 4. Attempt to find a valid spawn point within the spans
if (context.selectRandomSpawnSpan()) {
    boolean placed = false;
    if (/* some condition */ && context.isOnSolidGround()) {
        placed = true;
    } else if (/* another condition */ && context.isInWater(1.0f)) {
        placed = true;
    }

    // 5. Final validation
    if (placed && context.canSpawn() == SpawnTestResult.TEST_OK) {
        // Success! Get position and rotation
        Vector3d position = context.newPosition();
        Vector3f rotation = context.newRotation();
        // ... create the entity ...
    }
}

// 6. CRITICAL: Release the context to prevent memory leaks
context.release();
```

### Anti-Patterns (Do NOT do this)
-   **State Mismanagement:** Calling methods out of their logical order will result in an IllegalStateException. For example, calling `canSpawn` before `setSpawnable` and `selectRandomSpawnSpan` is invalid because the context lacks the necessary information (model dimensions and a target Y-level).
-   **Forgetting to Release:** Failing to call `release()` or `releaseFull()` after the spawning operation is complete. This will pin the World and WorldChunk objects in memory, causing a severe memory leak.
-   **Concurrent Modification:** Sharing and modifying a single SpawningContext instance across multiple threads. This will lead to race conditions and corrupt its internal state, producing unpredictable and erroneous spawning behavior.

## Data Pipeline
The SpawningContext processes world data to produce a viable spawn location. It does not transform data in a traditional pipeline sense, but rather refines a search space until a single point is validated.

> **Flow:**
> 1.  **Input:** A high-level spawner provides an `ISpawnableWithModel` (the "what") and a `WorldChunk` with (X, Z) coordinates (the "where").
> 2.  **Column Analysis:** The `setColumn` method consumes raw block and fluid data for the vertical column. It computes a list of `SpawnSpan` objects, representing valid vertical pockets of space.
> 3.  **Span Selection:** `selectRandomSpawnSpan` chooses one span and calculates detailed environmental properties like `groundLevel`, `waterLevel`, and `airHeight`.
> 4.  **Placement Strategy:** Methods like `isOnSolidGround` or `isInWater` use the selected span's properties to calculate a precise floating-point `ySpawn` coordinate.
> 5.  **Validation:** The `canSpawn` method takes the final computed position (`xSpawn`, `ySpawn`, `zSpawn`) and the entity's `Model` and passes them to the `CollisionModule`.
> 6.  **Output:** The `CollisionModule` returns a `SpawnTestResult`. If successful, the context holds the final, validated spawn position and rotation, ready to be retrieved via `newPosition()` and `newRotation()`.

