---
description: Architectural reference for SnapshotBuffer
---

# SnapshotBuffer

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Transient Component

## Definition
```java
// Signature
public class SnapshotBuffer implements Component<EntityStore> {
```

## Architecture & Concepts
The SnapshotBuffer is a fundamental component within the server-side Entity-Component-System (ECS) architecture, specifically designed for state management. It functions as a high-performance, fixed-size **circular buffer** (or ring buffer) that records an entity's historical states over a series of game ticks.

Its primary role is to enable server-authoritative features that require looking into the recent past, such as:
- **Lag Compensation:** Allowing the server to rewind an entity's state to a specific tick to validate a client's action (e.g., a weapon hit).
- **State Reconciliation:** Providing historical data to validate or correct client-side predictions.
- **Network Replication:** Buffering recent states to handle packet loss or to send historical data to newly connected clients.

Each element stored in the buffer is an EntitySnapshot, a lightweight data object containing an entity's position and rotation at a discrete point in time (a specific tick). The circular nature of the buffer ensures a constant memory footprint, overwriting the oldest snapshots as new ones are added.

## Lifecycle & Ownership
- **Creation:** A SnapshotBuffer is not instantiated directly. It is created and attached to a server-side Entity by the EntityModule when the entity is spawned or when the component is explicitly added. The buffer's initial size is configured via the `resize` method, typically during entity initialization.
- **Scope:** The lifecycle of a SnapshotBuffer is strictly bound to the lifecycle of its parent Entity. It persists as long as the entity exists in the world.
- **Destruction:** The component is marked for garbage collection and its memory is reclaimed when its parent Entity is destroyed and removed from the EntityStore. There is no manual destruction method.

## Internal State & Concurrency
- **State:** This component is highly mutable and stateful. Its core state consists of an array of EntitySnapshot objects and a set of integer indices (`currentIndex`, `currentTickIndex`, `oldestTickIndex`) that manage the write head and the valid range of the circular buffer. The internal array acts as a cache of historical entity states.

- **Thread Safety:** **This class is not thread-safe.** All public methods that modify its internal state (storeSnapshot, resize) or read from it (getSnapshot, getSnapshotClamped) are unsynchronized. Concurrent access from multiple threads will lead to race conditions, index corruption, and inconsistent state.

    **WARNING:** All interactions with a SnapshotBuffer instance must be synchronized externally, typically by confining its usage to the single-threaded server game loop responsible for ticking the parent entity.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSnapshotClamped(int tickIndex) | EntitySnapshot | O(1) | Retrieves the snapshot for a given tick. If the requested tick is older than the oldest available snapshot, it returns the oldest one. Throws IllegalArgumentException for future ticks. |
| getSnapshot(int tickIndex) | EntitySnapshot | O(1) | Retrieves the snapshot for a given tick. Returns null if the tick is older than the oldest available snapshot. Throws IllegalArgumentException for future ticks. |
| storeSnapshot(int tickIndex, Vector3d pos, Vector3f rot) | void | O(1) | Writes a new snapshot into the buffer for the specified tick. This advances the internal indices of the circular buffer, potentially overwriting the oldest snapshot. |
| resize(int newLength) | void | O(N) | Re-allocates the internal snapshot array to a new size. This is a destructive and expensive operation that resets the buffer's state. |
| isInitialized() | boolean | O(1) | Returns true if at least one snapshot has been stored. |
| clone() | Component | O(K) | Creates a deep copy of the buffer, where K is the number of valid snapshots. It iterates through and stores each valid snapshot into the new instance. |

## Integration Patterns

### Standard Usage
The component is typically managed by an entity processing system within the server's main loop. On each tick, the system updates the entity's state and then records it in the buffer.

```java
// Within a server-side entity processing system
Entity entity = ...;
int currentServerTick = server.getTick();

// Assume the entity has been updated for this tick
Vector3d newPosition = entity.getPosition();
Vector3f newRotation = entity.getRotation();

// Retrieve the component and store the new state
SnapshotBuffer buffer = entity.getComponent(SnapshotBuffer.class);
if (buffer != null) {
    buffer.storeSnapshot(currentServerTick, newPosition, newRotation);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new SnapshotBuffer()`. The component must be managed by the ECS framework to be correctly associated with an entity.
- **Concurrent Modification:** Do not access a SnapshotBuffer from asynchronous tasks or other threads while the main game loop is running. This will corrupt the buffer's internal state. All access must be serialized.
- **Frequent Resizing:** Calling `resize` repeatedly is a significant performance anti-pattern. The buffer size should be determined once during entity initialization and remain constant. Resizing causes heap allocation, array copying, and a complete reset of the historical data.

## Data Pipeline
The SnapshotBuffer acts as a temporary, in-memory cache for an entity's recent history. It sits between the state generation logic (physics/gameplay simulation) and state consumption logic (networking/validation).

> Flow:
> Entity Simulation Logic -> **SnapshotBuffer.storeSnapshot()** -> [Buffer holds state for N ticks] -> Lag Compensation System -> **SnapshotBuffer.getSnapshot(tick)** -> Hit Validation Logic

