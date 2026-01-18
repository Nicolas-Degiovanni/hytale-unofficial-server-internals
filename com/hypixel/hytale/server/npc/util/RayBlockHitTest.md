---
description: Architectural reference for RayBlockHitTest
---

# RayBlockHitTest

**Package:** com.hypixel.hytale.server.npc.util
**Type:** Pooled Procedure Object

## Definition
```java
// Signature
public class RayBlockHitTest implements BlockIterator.BlockIteratorProcedure {
```

## Architecture & Concepts

The RayBlockHitTest is a highly-optimized, stateful procedure object designed for performing targeted raycasts against the block world on the server. Its primary purpose is to determine if an entity, typically an NPC, has a line of sight to a block belonging to a specific BlockSet.

Architecturally, this class implements the Strategy Pattern. It encapsulates the *logic* of what to do at each step of a raycast, while the core `BlockIterator` class handles the complex *algorithm* of traversing a ray through the 3D block grid. This separation of concerns keeps the core traversal algorithm generic and reusable.

The most critical architectural feature is its management via a `ThreadLocal` pool. This design pattern is employed to eliminate object allocation and garbage collection overhead in performance-sensitive server ticks. By reusing a single RayBlockHitTest instance per thread, the system avoids the significant cost of creating and destroying these objects for every raycast, which can occur thousands of time per second on a busy server.

## Lifecycle & Ownership

The lifecycle of a RayBlockHitTest instance is intrinsically tied to the `ThreadLocal` pool and the thread that owns it. Understanding this is critical to prevent state leakage and ensure system stability.

-   **Creation:** An instance is lazily instantiated by the `ThreadLocal.withInitial` factory the first time `THREAD_LOCAL.get()` is called on a given thread. It is not created via a public constructor.
-   **Scope:** The object instance persists for the entire lifetime of the thread. It is intended to be acquired, used, and cleared within a single logical operation. **It must never be passed between threads.**
-   **Destruction:** The object is only eligible for garbage collection when its owning thread terminates. The `clear` method does not destroy the object; it resets its internal state, making it ready for the next operation on the same thread.

**WARNING:** Failure to call `clear` in a `finally` block after use is a critical bug. It will cause state from a previous raycast to leak into the next, leading to highly unpredictable behavior.

## Internal State & Concurrency

-   **State:** This object is highly mutable. Its internal fields (`world`, `chunk`, `origin`, `direction`, `hitPosition`) are populated by the `init` method and modified during the `run` operation. The state is only valid for the duration of a single, complete raycast operation.
-   **Thread Safety:** **This class is not thread-safe.** It is explicitly designed for thread-confinement. The `ThreadLocal` wrapper is the mechanism that ensures safety by preventing shared access. Any attempt to manually share an instance of this class between threads will result in state corruption and severe, difficult-to-diagnose race conditions.

## API Surface

The public contract is designed for a sequential, three-step process: initialize, run, and clear.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(ref, blockSet, pitch, accessor) | boolean | O(1) | Configures the object for a new raycast. Acquires entity data to set the ray's origin and direction. Returns false if the entity's starting chunk is not loaded. |
| run(range) | boolean | O(N) | Executes the raycast up to the specified range, where N is the number of blocks traversed. Returns true if a valid block was hit, false otherwise. |
| getHitPosition() | Vector3d | O(1) | Retrieves the world coordinates of the block that was hit. Returns Vector3d.MIN if no hit occurred. |
| clear() | void | O(1) | Resets all internal state, preparing the object for reuse. Nullifies references to world and chunk. |

## Integration Patterns

### Standard Usage

The canonical usage pattern involves acquiring the thread-local instance, using a `try-finally` block to guarantee cleanup, and executing the operation.

```java
// Acquire the thread-local instance for this operation
RayBlockHitTest hitTest = RayBlockHitTest.THREAD_LOCAL.get();

try {
    // 1. Initialize with entity data and target block set
    boolean canRun = hitTest.init(entityRef, blockSetId, entityPitch, accessor);

    if (canRun) {
        // 2. Execute the raycast with a maximum range
        boolean hit = hitTest.run(64.0);

        if (hit) {
            // 3. Process the result
            Vector3d position = hitTest.getHitPosition();
            // ... do something with the hit position
        }
    }
} finally {
    // 4. CRITICAL: Always clear the state for the next user on this thread
    hitTest.clear();
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not call `new RayBlockHitTest()`. This completely bypasses the `ThreadLocal` pooling mechanism, defeating the performance optimization and leading to unnecessary garbage collection.
-   **Omitting `clear`:** Forgetting to call `clear()` in a `finally` block is the most common and dangerous error. State from the previous operation (e.g., a successful `hitPosition`) will persist, causing subsequent, unrelated raycasts on the same thread to return false positives.
-   **Storing the Instance:** Do not store the instance returned by `THREAD_LOCAL.get()` in a long-lived object field. It should be acquired and released within the scope of a single method.

## Data Pipeline

The RayBlockHitTest acts as a stateful processor within a larger data flow, transforming an entity's state into a world-space block coordinate.

> Flow:
> Server Tick Logic -> **RayBlockHitTest.THREAD_LOCAL.get()** -> `init()` with Entity Transform -> `run()` -> BlockIterator Core -> `accept()` callback for each block -> **Internal `hitPosition` updated** -> Server Tick Logic reads result

