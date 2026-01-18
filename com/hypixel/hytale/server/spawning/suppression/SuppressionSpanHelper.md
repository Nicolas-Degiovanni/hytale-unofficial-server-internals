---
description: Architectural reference for SuppressionSpanHelper
---

# SuppressionSpanHelper

**Package:** com.hypixel.hytale.server.spawning.suppression
**Type:** Transient

## Definition
```java
// Signature
public class SuppressionSpanHelper {
```

## Architecture & Concepts

The SuppressionSpanHelper is a high-performance, stateful utility class designed to optimize the server's mob spawning algorithm. Its primary function is to process and query vertical regions (spans on the Y-axis) within a world chunk where entity spawning is disallowed.

This class acts as a crucial optimization layer within the spawning pipeline. Instead of the spawning system iterating through a potentially large and redundant list of suppression rules for every potential spawn location, it uses this helper to first pre-process and merge these rules into a minimal, non-overlapping set of "suppression spans".

The core architectural pattern is **data pre-processing and stateful iteration**. The helper is first initialized with raw suppression data for a specific context (a chunk and an entity role). It then provides fast query methods that leverage this pre-processed state. To minimize garbage collection pressure during intensive spawning cycles, it utilizes a thread-local object pool for its internal Span objects.

## Lifecycle & Ownership

-   **Creation:** An instance of SuppressionSpanHelper is expected to be created on-demand by a higher-level spawning manager, likely at the beginning of a spawning evaluation for a specific world chunk. It is a short-lived, single-use object.

-   **Scope:** The object's state is valid only for the duration of a single, complete spawning operation. Its lifecycle is tightly bound to the scope of the method that creates it.

-   **Destruction:** The object itself is eligible for garbage collection once it falls out of scope. However, it is **critical** that the `reset` method is called before the object is discarded. This method returns internal data structures to a thread-local object pool. Failure to call `reset` will result in a memory leak within the pool, persisting for the lifetime of the worker thread.

## Internal State & Concurrency

-   **State:** This class is highly mutable and stateful. It maintains an internal list of `optimisedSuppressionSpans` and a cursor, `currentSpanIndex`, to track progress during queries. The state is populated by `optimiseSuppressedSpans` and consumed by the `adjustSpawnRange` methods.

-   **Thread Safety:** **This class is not thread-safe and must not be shared across threads.** Its mutable internal state, including the iteration cursor, would be immediately corrupted by concurrent access. The use of a `ThreadLocal` object pool reinforces this design; each server worker thread is expected to manage its own instances of SuppressionSpanHelper.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| optimiseSuppressedSpans(roleIndex, entry) | void | O(N) | Initializes the helper by processing and merging raw suppression spans from a ChunkSuppressionEntry. N is the number of raw spans. This method must be called once before any `adjust` methods. |
| adjustSpawnRangeMin(min) | int | Amortized O(1) | Given a minimum Y-level, returns an adjusted minimum that skips any suppression span it falls within. Advances an internal cursor. |
| adjustSpawnRangeMax(min, max) | int | O(1) | Given a potential spawn range, returns an adjusted maximum Y-level to avoid intersecting with the current suppression span. |
| reset() | void | O(M) | Clears all internal state and returns allocated objects to a thread-local pool. M is the number of *optimised* spans. **This must be called after use.** |

## Integration Patterns

### Standard Usage

The intended use is a three-phase pattern: initialize, query, and reset. This pattern ensures correct behavior and prevents resource leaks. The `reset` call should always be placed in a `finally` block.

```java
// 1. Obtain a helper instance for a single spawning operation
SuppressionSpanHelper helper = new SuppressionSpanHelper();

try {
    // 2. INITIALIZE with data for the current context
    ChunkSuppressionEntry suppressionData = getSuppressionDataForChunk(chunkPos);
    helper.optimiseSuppressedSpans(entityRoleIndex, suppressionData);

    // 3. QUERY in a loop to find valid spawn locations
    for (int y = world.getMinY(); y < world.getMaxY(); y++) {
        int adjustedMinY = helper.adjustSpawnRangeMin(y);
        if (adjustedMinY > y) {
            // The original 'y' was in a suppressed span; jump ahead
            y = adjustedMinY - 1; // -1 to account for loop increment
            continue;
        }
        // ... proceed with spawning logic at this valid 'y' ...
    }
} finally {
    // 4. RESET to release resources back to the thread-local pool
    helper.reset();
}
```

### Anti-Patterns (Do NOT do this)

-   **Forgetting Reset:** The most critical anti-pattern is failing to call `reset`. This will leak `Span` objects into the `ThreadLocal` pool, causing a slow memory leak that persists for the life of the thread.
-   **Reusing Without Reset:** Reusing a helper instance for a new spawning operation without first calling `reset` will cause it to operate on stale data, leading to incorrect and unpredictable spawning suppression.
-   **Sharing Across Threads:** Accessing a single helper instance from multiple threads will cause race conditions on its internal state, leading to crashes or incorrect game logic.
-   **Incorrect Call Order:** Calling `adjustSpawnRangeMin` or `adjustSpawnRangeMax` before `optimiseSuppressedSpans` has been called will result in no-op behavior, as the internal list of spans will be empty. This can lead to silent failures where spawning suppression does not work as intended.

## Data Pipeline

The helper transforms raw, verbose suppression data into a compact, queryable format that directly influences the final selection of a spawn location.

> Flow:
> ChunkSuppressionEntry (Raw Spans) -> **SuppressionSpanHelper.optimiseSuppressedSpans** -> Internal List of Merged Spans -> **SuppressionSpanHelper.adjust...** -> Adjusted Y-Coordinate -> Spawning Algorithm

