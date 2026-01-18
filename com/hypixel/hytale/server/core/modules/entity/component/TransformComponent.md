---
description: Architectural reference for TransformComponent
---

# TransformComponent

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Data Component

## Definition
```java
// Signature
public class TransformComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The TransformComponent is a fundamental data component within the server-side Entity-Component-System (ECS) architecture. Its sole responsibility is to define an entity's spatial state within the game world, specifically its precise position and orientation.

As a core data provider, this component does not contain complex logic. Instead, it serves as the authoritative source of truth for numerous other systems:
- **Physics System:** Reads and writes position and rotation data each simulation tick.
- **AI System:** Modifies position and rotation to execute navigation and behavior logic.
- **Network System:** Reads the state to replicate entity movements to clients.
- **Persistence System:** Serializes the component's state to disk when a world chunk is saved.

The component's design directly integrates with the world's spatial partitioning system. The inclusion of a `chunkRef` establishes a direct link between the entity and the `WorldChunk` it currently occupies. This is critical for efficient region-based queries, network culling, and world saving operations. The `sentTransform` field is an optimization for the network layer, caching the last state sent to clients to prevent the transmission of redundant data when an entity is stationary.

## Lifecycle & Ownership
- **Creation:** A TransformComponent is instantiated and attached to an entity when that entity is spawned into the world. This is typically managed by the `EntityModule` or a higher-level entity factory, which ensures the component is initialized with a valid starting position and rotation.
- **Scope:** The component's lifecycle is strictly bound to its parent entity. It persists as long as the entity exists in the world.
- **Destruction:** The component is marked for garbage collection when its parent entity is despawned or removed from the world. No manual cleanup is required.

## Internal State & Concurrency
- **State:** The internal state is highly mutable. The `position` and `rotation` vectors are expected to be modified frequently, often multiple times per game tick by various systems. The component also caches its containing chunk reference (`chunkRef`) and the last-sent network state (`sentTransform`).

- **Thread Safety:** This component is **not thread-safe**. It contains no internal locking or synchronization mechanisms. The engine's architecture must guarantee that all modifications to a single TransformComponent instance are serialized. Unsynchronized, concurrent access from multiple threads (e.g., a physics thread and an AI thread modifying position simultaneously) will lead to race conditions, corrupted state, and unpredictable behavior. All interactions must be coordinated through a job system with defined dependencies or be confined to a single, main game-loop thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | ComponentType | O(1) | Static method to retrieve the unique type identifier for this component class. |
| teleportPosition(Vector3d) | void | O(1) | Sets the entity's position, with safety checks to ignore NaN values. |
| teleportRotation(Vector3f) | void | O(1) | Sets the entity's rotation, with safety checks to ignore NaN values. |
| markChunkDirty(...) | void | O(1) | Notifies the world storage system that the chunk containing this entity has been modified and requires saving. |
| setChunkLocation(...) | void | O(1) | Updates the internal reference to the `WorldChunk` that currently contains this entity. |

## Integration Patterns

### Standard Usage
A system retrieves the TransformComponent from an entity to read or modify its spatial state. Logic should always be performed through the entity's component accessor.

```java
// Example: A system that moves an entity upwards
void process(Entity entity) {
    TransformComponent transform = entity.getComponent(TransformComponent.getComponentType());
    if (transform != null) {
        Vector3d currentPos = transform.getPosition();
        currentPos.add(0, 0.1, 0); // Modify the position vector directly
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new TransformComponent()`. Components must be created and managed by the entity system (e.g., `entity.addComponent(...)`) to ensure they are correctly registered and tracked within the world.
- **Unsynchronized Cross-Thread Modification:** Never modify a TransformComponent from one thread while another thread might be reading or writing to it. This will cause severe and difficult-to-debug race conditions. All access must be synchronized by the parent engine architecture.
- **Position Change Without Spatial Update:** Modifying the `position` vector moves the entity logically, but the world's spatial partitioning system (which uses chunks) will not be aware of the change. A proper move operation must also update the entity's chunk registration, typically by notifying a higher-level `EntityManager` which will then call `setChunkLocation`. Failure to do so will result in the entity becoming "lost" to chunk-based queries.

## Data Pipeline
The TransformComponent acts as a central data bus for an entity's spatial information. It is constantly being written to by gameplay systems and read from by infrastructure systems.

> **Server Tick Input Flow:**
> AI System / Physics Engine -> **TransformComponent.setPosition()** / **.setRotation()** -> World Spatial System -> **TransformComponent.setChunkLocation()**

> **Server Tick Output Flow:**
> **TransformComponent** -> Network System (builds replication packets) -> Client
> **TransformComponent** -> Persistence System (serializes via CODEC) -> Disk Storage

