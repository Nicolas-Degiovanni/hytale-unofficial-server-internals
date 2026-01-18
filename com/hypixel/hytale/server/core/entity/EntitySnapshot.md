---
description: Architectural reference for EntitySnapshot
---

# EntitySnapshot

**Package:** com.hypixel.hytale.server.core.entity
**Type:** Transient Data Transfer Object (DTO)

## Definition
```java
// Signature
public class EntitySnapshot {
```

## Architecture & Concepts
The EntitySnapshot is a fundamental data structure that represents the spatial state of an entity at a discrete moment in time. It serves as a lightweight, self-contained Data Transfer Object (DTO) designed to decouple the live, mutable state of an entity from systems that require a stable, point-in-time view.

Its primary role is to capture an entity's position and rotation, preventing race conditions and data inconsistencies when this information is passed between major engine subsystems. Common use cases include:

-   **Network Replication:** The server's game loop creates snapshots to serialize and transmit entity state to clients.
-   **Interpolation & Prediction:** Both client and server can store a history of snapshots to smoothly render entity movement or perform client-side prediction.
-   **Physics & Collision:** The physics engine may use snapshots to store an entity's previous state for resolving collisions or calculating movement vectors.

Architecturally, this class is designed for high-frequency allocation and deallocation. The presence of an `init` method strongly suggests its intended use within an object pooling system to mitigate garbage collection overhead in performance-critical loops.

## Lifecycle & Ownership
-   **Creation:** An EntitySnapshot is typically instantiated by a higher-level system that manages game state, such as an `EntityManager` or a `WorldTicker`. It is created whenever a system needs to freeze and transport an entity's state. In high-performance contexts, instances are not created with `new` but are instead acquired from a dedicated object pool.
-   **Scope:** The lifetime of an EntitySnapshot is extremely short. It is designed to be ephemeral, existing only for the duration of a single operation, such as the processing of one game tick or the construction of one network packet.
-   **Destruction:** Once its data has been consumed by the target system, the snapshot is immediately eligible for garbage collection. If managed by an object pool, the instance is returned to the pool for reuse via a `release` method (not shown in this class, but implied by the architecture).

## Internal State & Concurrency
-   **State:** The class is **mutable**. Although the references to the internal `position` and `bodyRotation` vectors are final, the vector objects themselves are mutable. The `init` method directly modifies the internal state of the object, allowing a single instance to be recycled to represent different snapshots over time.
-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms. It is designed to be created, populated, and read within a single thread or passed between threads via safe hand-off mechanisms like a producer-consumer queue. Unsynchronized access from multiple threads will lead to data corruption and non-deterministic behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(position, bodyRotation) | void | O(1) | Re-initializes the snapshot with new state. This is the primary method for populating a recycled instance from an object pool. |
| getPosition() | Vector3d | O(1) | Returns a **direct reference** to the internal position vector. See Anti-Patterns below. |
| getBodyRotation() | Vector3f | O(1) | Returns a **direct reference** to the internal body rotation vector. See Anti-Patterns below. |

## Integration Patterns

### Standard Usage
The typical use case involves capturing an entity's current state for processing by another system.

```java
// In a game loop, capture the state of an entity for networking
Entity entity = world.getEntity(entityId);
EntitySnapshot snapshot = new EntitySnapshot(entity.getPosition(), entity.getBodyRotation());

// Pass the snapshot to the network serialization layer
networkSystem.sendState(snapshot);
```

### Anti-Patterns (Do NOT do this)
-   **Modifying Returned Vectors:** The `getPosition` and `getBodyRotation` methods return direct references to the internal state, not copies. Modifying the returned vector will corrupt the state of the snapshot. This is a critical performance optimization that places responsibility on the caller.

    ```java
    // DO NOT DO THIS
    EntitySnapshot snapshot = getSnapshot();
    Vector3d pos = snapshot.getPosition();
    pos.setX(100.0); // This corrupts the snapshot's internal state!
    ```

-   **State Bleeding:** Reusing a snapshot instance without re-initializing it. If a snapshot is held for longer than a single operation, its data will become stale and irrelevant.

## Data Pipeline
The EntitySnapshot acts as a data shuttle, most commonly moving state from the core game simulation to outbound systems like networking or rendering.

> Flow:
> Game Loop Tick -> Entity State Update -> **EntitySnapshot (State Capture)** -> Network Serialization -> Outbound Packet Queue

