---
description: Architectural reference for SpawnMarkerBlockState
---

# SpawnMarkerBlockState

**Package:** com.hypixel.hytale.server.spawning.blockstates
**Type:** Transient State Object

## Definition
```java
// Signature
public class SpawnMarkerBlockState extends BlockState {
```

## Architecture & Concepts
The SpawnMarkerBlockState is a specialized implementation of BlockState that serves as a bridge between a static block in the world and a dynamic, managed entity. Its primary function is to anchor a SpawnMarker entity to a specific block coordinate, enabling designers to place spawn points and other logical markers directly within the world geometry.

This class embodies a separation of concerns between configuration and runtime state:
1.  **Configuration (Inner class `Data`):** Defines the *type* of marker to spawn (via a string asset reference) and its positional offset relative to the block. This data is loaded from asset files and is immutable at runtime.
2.  **Runtime State (The `SpawnMarkerBlockState` instance):** Represents the live state of a specific block in a loaded world chunk. It holds a persistent reference (PersistentRef) to the actual spawned marker entity and manages a timeout mechanism to detect if that entity is ever lost or destroyed.

The use of a Codec for both the configuration and runtime state is critical. It allows the engine to serialize this state for world saves and deserialize it upon loading, ensuring that the link between the block and its associated marker entity persists across server sessions.

## Lifecycle & Ownership
-   **Creation:** An instance is created by the world engine when a block of this type is placed or generated. Initially, its spawnMarkerReference will be null. A dedicated spawning system is responsible for observing these block states, creating the corresponding SpawnMarker entity, and then populating the reference using setSpawnMarkerReference.
-   **Scope:** The object's lifetime is strictly tied to the block it represents within a loaded world chunk. It persists as long as the block exists and the chunk is held in memory.
-   **Destruction:** The object is destroyed and garbage collected under two conditions:
    1.  The block in the world is broken or replaced.
    2.  The world chunk containing the block is unloaded from server memory.

## Internal State & Concurrency
-   **State:** This object is highly mutable. Its core purpose is to track the live reference to an entity and a countdown timer (markerLostTimeout). These values are expected to change during the game loop as part of the spawning system's update cycle.
-   **Thread Safety:** **This class is not thread-safe.** As a component of the world state, it must only be accessed and modified from the main server thread (the world tick thread). Unsynchronized access from other threads will lead to race conditions, data corruption, and server instability. All interactions, particularly calls to tickMarkerLostTimeout, must be serialized within the main game loop.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSpawnMarkerReference() | PersistentRef | O(1) | Retrieves the persistent reference to the associated marker entity. |
| setSpawnMarkerReference(ref) | void | O(1) | Sets the reference to the marker entity. Typically called once by the spawning system after entity creation. |
| refreshMarkerLostTimeout() | void | O(1) | Resets the internal timeout countdown. This should be called each tick that the associated entity is confirmed to be valid. |
| tickMarkerLostTimeout(dt) | boolean | O(1) | Decrements the timeout by the delta time (dt). Returns true if the timeout has expired, signaling that the marker entity has been lost. |

## Integration Patterns

### Standard Usage
The intended pattern involves a higher-level management system that interacts with this state object as part of the main server tick.

```java
// Within a SpawningSystem or similar manager
for (Block block : world.getBlocksOfType(SpawnMarkerBlock.class)) {
    SpawnMarkerBlockState state = block.getState(SpawnMarkerBlockState.class);
    
    // Check if the referenced entity is still valid
    Entity marker = world.findEntity(state.getSpawnMarkerReference());

    if (marker != null) {
        state.refreshMarkerLostTimeout();
    } else {
        // If the entity is gone, tick the timeout
        // The return value signals that the block state should be cleaned up
        if (state.tickMarkerLostTimeout(deltaTime)) {
            // Logic to handle a lost marker, e.g., respawn it or log a warning
            cleanupLostMarker(block);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new SpawnMarkerBlockState()`. The world engine is solely responsible for creating and managing the lifecycle of block state objects.
-   **Asynchronous Modification:** Never call `tickMarkerLostTimeout` or `setSpawnMarkerReference` from a separate thread. All state changes must be synchronized with the world tick.
-   **Ignoring Timeout:** Failing to call `tickMarkerLostTimeout` will prevent the system from ever detecting and cleaning up orphaned markers, leading to a permanent state mismatch between the block and the entity world.

## Data Pipeline
This component participates in two distinct data flows: initial configuration loading and runtime state management.

> **Configuration Flow:**
> Asset File (`.json`) -> Hytale Asset Loader -> **SpawnMarkerBlockState.Data** (Deserialized) -> BlockType Definition

> **Runtime State Flow:**
> World Tick -> Spawning System -> **SpawnMarkerBlockState**.tickMarkerLostTimeout() -> (if true) -> Cleanup/Respawn Logic

