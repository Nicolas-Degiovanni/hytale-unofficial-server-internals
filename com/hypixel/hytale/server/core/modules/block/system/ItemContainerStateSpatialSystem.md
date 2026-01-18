---
description: Architectural reference for ItemContainerStateSpatialSystem
---

# ItemContainerStateSpatialSystem

**Package:** com.hypixel.hytale.server.core.modules.block.system
**Type:** Transient

## Definition
```java
// Signature
public class ItemContainerStateSpatialSystem extends SpatialSystem<ChunkStore> {
```

## Architecture & Concepts
The **ItemContainerStateSpatialSystem** is a specialized component within the server's Entity Component System (ECS) framework. Its primary function is to act as a translator, converting the abstract, chunk-local block coordinates of item containers into absolute, global world coordinates.

This system is a concrete implementation of **SpatialSystem**, a core engine abstraction for managing the position and orientation of entities in the game world. It exclusively targets entities possessing an **ItemContainerState** component, as defined by its static **QUERY** field. This precise targeting ensures that the system only performs work on relevant block entities, such as chests or barrels.

The core logic resides in the **getPosition** method, which calculates a **Vector3d** world position from a block's reference within a **BlockChunk**. This system is the definitive authority for determining *where* an item container is located in 3D space, enabling other systems like physics, rendering, and AI to interact with it correctly.

A key performance feature is the conditional execution within the **tick** method. It consults the **BlockStateInfoNeedRebuild** resource before proceeding with a full update. This mechanism prevents costly, redundant spatial calculations on every server tick, only running when the underlying block state data has actually changed.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's ECS framework during the initialization of the **BlockModule**. It is not created directly by user code.
-   **Scope:** The instance persists for the entire server session, managed by the ECS system scheduler.
-   **Destruction:** Decommissioned and garbage collected when the server shuts down or the parent **BlockModule** is unloaded.

## Internal State & Concurrency
-   **State:** This class is fundamentally stateless. It does not maintain any mutable instance fields. All computations are performed on data passed in via method parameters, primarily the **ArchetypeChunk** and **Store**.
-   **Thread Safety:** **This system is not thread-safe.** It is designed to be executed exclusively on the main server thread as part of the synchronous game loop. Its methods read from shared ECS data structures (**Store**, **ArchetypeChunk**) which are not protected for concurrent access. Unsynchronized calls from other threads will lead to data corruption and server instability.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, systemIndex, store) | void | O(N) | Executes the spatial update logic for all matching components. N is the number of entities with a dirty **ItemContainerState**. |
| getPosition(archetypeChunk, index) | Vector3d | O(1) | Calculates the absolute world position for a single block entity. Returns null if the chunk reference is invalid. |
| getQuery() | Query | O(1) | Returns the static ECS query used to identify target entities for this system. |

## Integration Patterns

### Standard Usage
This system is not intended to be invoked directly. It is automatically registered and managed by the **BlockModule**. Other game systems that require the location of an item container should query the engine's central spatial database, which is populated by this system's calculations.

```java
// PSEUDOCODE: How the engine uses this system internally
// This is for illustration only. Do not replicate.

// 1. During module initialization, the system is registered.
ecsFramework.registerSystem(new ItemContainerStateSpatialSystem(...));

// 2. During the main game loop, the framework calls tick().
// The framework is responsible for passing the correct arguments.
activeSystem.tick(deltaTime, index, world.getStore());
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call **new ItemContainerStateSpatialSystem()**. The ECS framework is the sole owner and manager of its lifecycle. Manually creating an instance will result in a non-functional, disconnected system.
-   **Manual Invocation:** Do not call the **tick** or **getPosition** methods directly. Bypassing the ECS scheduler can break the conditional update logic and lead to severe performance degradation or stale spatial data.
-   **Multi-threaded Access:** Never access this system from any thread other than the main server thread. This will cause critical race conditions.

## Data Pipeline
The primary data flow involves the transformation of chunk-relative coordinates into world-absolute coordinates.

> Flow:
> Block Entity Change -> **BlockStateInfoNeedRebuild** flag is set -> **ItemContainerStateSpatialSystem.tick** is triggered -> **getPosition** is called for the entity -> Reads **BlockChunk** coordinates and local block index -> Performs bitwise shifts and additions -> Returns **Vector3d** -> Engine's spatial database (e.g., Octree) is updated

