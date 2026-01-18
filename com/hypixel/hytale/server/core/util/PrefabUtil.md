---
description: Architectural reference for PrefabUtil
---

# PrefabUtil

**Package:** com.hypixel.hytale.server.core.util
**Type:** Utility

## Definition
```java
// Signature
public class PrefabUtil {
```

## Architecture & Concepts
PrefabUtil is a high-level, stateless orchestrator for server-side world modification. It serves as the primary engine API for all operations involving prefabs, which are pre-designed structural templates of blocks and entities. This class acts as a critical bridge between the abstract data representation of a structure, encapsulated in an IPrefabBuffer, and the concrete, low-level world state managed by the World and its associated stores.

Its core responsibility is to translate the contents of a prefab buffer into tangible changes within the game world. This is not a simple block-by-block iteration; PrefabUtil manages a complex sequence of operations:

1.  **Performance Optimization:** Before any modification, it establishes a LocalCachedChunkAccessor. This accessor pre-fetches and caches all world chunks within the prefab's bounding box, dramatically reducing I/O latency and redundant lookups during the paste operation. This is a foundational performance pattern for any large-scale world edit.
2.  **Coordinate Space Transformation:** It applies positional offsets and rotational transformations to every block and entity within the prefab, correctly placing them in world space relative to the target origin and yaw.
3.  **Event-Driven Integration:** The paste process is wrapped by PrefabPasteEvent dispatches. A "start" event is fired before modification, and an "end" event is fired after. This allows other game systems, such as quest managers or zone controllers, to inspect, modify, or even cancel a paste operation, providing a robust hook for gameplay logic.
4.  **Stateful Block Placement:** Beyond simply setting block IDs, it handles the placement of complex block states (via Holder objects) and physics properties like support values.
5.  **Entity Instantiation:** It clones entity templates from the prefab, correctly transforms their position and rotation, and injects them into the world's EntityStore, ensuring they are properly initialized and tracked by the entity component system.

WARNING: While this is a utility class, its methods are heavyweight operations that can cause significant server load and must be used judiciously.

## Lifecycle & Ownership
- **Creation:** As a static utility class, PrefabUtil is never instantiated. Its methods are accessed statically.
- **Scope:** The class and its methods are available for the entire lifecycle of the server application, loaded by the JVM class loader at startup.
- **Destruction:** Not applicable. The class is unloaded when the server application terminates.

## Internal State & Concurrency
- **State:** PrefabUtil is fundamentally stateless. All required state (World, IPrefabBuffer, position) is passed as arguments to its methods. The single exception is a private static AtomicInteger, PREFAB_ID_SOURCE, used to generate unique, thread-safe identifiers for paste operations.
- **Thread Safety:** The methods themselves are **not** thread-safe with respect to the World object. The World has a strict single-threaded execution model. Calling a method like paste from an asynchronous thread is extremely dangerous and will lead to world corruption.

    The class contains internal logic (`!world.isInThread()`) to delegate certain sub-tasks back to the main world thread via CompletableFuture.runAsync. This is a defensive mechanism, not a license for off-thread invocation. Callers are responsible for ensuring that any invocation that modifies the world is executed on the correct world thread. The getNextPrefabId method is an exception and is fully thread-safe.

## API Surface
The primary API surface consists of high-level commands to query and manipulate the world using prefabs.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| paste(...) | void | O(N) | The primary operation. Pastes a prefab into the world. Throws IllegalArgumentException if required editor blocks are not defined. N is the number of blocks and entities in the buffer. |
| remove(...) | void | O(N) | The inverse of paste. Removes the blocks defined by a prefab from the world. N is the number of blocks in the buffer. |
| canPlacePrefab(...) | boolean | O(N) | A non-mutating pre-flight check to determine if a prefab can be placed without overwriting solid blocks. |
| prefabMatchesAtPosition(...) | boolean | O(N) | A non-mutating check to see if the world's current state at a given position perfectly matches the prefab's structure. |
| getNextPrefabId() | int | O(1) | Atomically increments and returns a unique ID for a prefab operation. Thread-safe. |

## Integration Patterns

### Standard Usage
The standard pattern involves acquiring a prefab buffer, defining a target location, and invoking the paste method from a system that operates on the main server thread.

```java
// Assume 'world' and 'prefabBuffer' are already acquired
// This code MUST be executed on the world's main thread.

Vector3i targetPosition = new Vector3i(100, 64, 250);
Rotation yaw = Rotation.Y_90;
Random random = world.getRandom();
ComponentAccessor<EntityStore> accessor = world.getEntityStore().getAccessor();

// Pre-flight check is recommended for non-destructive placement
if (PrefabUtil.canPlacePrefab(prefabBuffer, world, targetPosition, yaw, null, random, false)) {
    // Execute the paste operation
    PrefabUtil.paste(prefabBuffer, world, targetPosition, yaw, false, random, accessor);
}
```

### Anti-Patterns (Do NOT do this)
- **Off-Thread World Modification:** Never call `paste` or `remove` from a thread that is not the world's primary update thread. This will bypass engine safeguards and cause severe concurrency issues, including deadlocks and data corruption.
- **Ignoring Pre-Flight Checks:** For gameplay logic that respects existing structures, calling `paste` with `force=false` without first calling `canPlacePrefab` can lead to failed placements. The pre-flight check is more efficient for determining placement validity.
- **Large-Scale Synchronous Pasting:** Invoking `paste` for an extremely large prefab (e.g., 512x256x512) during a live game tick can cause a significant server stall. For such operations, consider chunk-by-chunk pasting spread across multiple ticks.

## Data Pipeline
PrefabUtil orchestrates a clear flow of data from an abstract representation to a concrete world state.

> Flow:
> IPrefabBuffer (In-memory structure) -> **PrefabUtil.paste** (Orchestrator) -> PrefabPasteEvent (Start) -> LocalCachedChunkAccessor (Chunk Caching) -> WorldChunk API (Block/Entity Mutation) -> PrefabPasteEvent (End) -> Modified World State

