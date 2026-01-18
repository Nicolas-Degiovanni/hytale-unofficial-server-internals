---
description: Architectural reference for NetworkSendableSpatialSystem
---

# NetworkSendableSpatialSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** System Component

## Definition
```java
// Signature
public class NetworkSendableSpatialSystem extends SpatialSystem<EntityStore> {
```

## Architecture & Concepts
The NetworkSendableSpatialSystem is a specialized implementation of the generic SpatialSystem, designed to integrate network-replicated entities into the server's spatial partitioning data structure. Its primary function is to act as a bridge between the Entity Component System (ECS) and the server's Area of Interest (AoI) or interest management framework.

This system exclusively targets entities that possess both a **TransformComponent** (defining their position, rotation, and scale) and a **NetworkId** (identifying them as entities that should be synchronized with clients). This filtering is achieved via its static QUERY, making it a highly focused and efficient component of the server's core loop.

By populating a spatial data structure (such as an octree or grid, managed by the parent SpatialSystem) with these network-relevant entities, other systems can perform highly efficient spatial queries. For example, the network serialization layer can query this system to find all entities within a certain radius of a player to determine which entity data packets need to be sent.

In essence, this class is the foundational layer for network culling and relevance determination on the server.

### Lifecycle & Ownership
- **Creation:** Instantiated once per world instance by the server's core `SystemScheduler` or an equivalent bootstrap mechanism during world initialization. It is not intended for manual creation.
- **Scope:** The lifecycle of a NetworkSendableSpatialSystem instance is bound to the lifecycle of the server world it manages. It persists for the entire duration of a game session.
- **Destruction:** The instance is marked for garbage collection when its associated world is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** This specific class is stateless. Its only field is a static final QUERY constant. However, its superclass, **SpatialSystem**, is highly stateful, containing the reference to and managing the underlying `SpatialResource` which holds the complex spatial data structure. This class provides the logic, while the parent class holds the state.
- **Thread Safety:** This system is **not thread-safe**. It is designed to be operated exclusively by the main server thread as part of the synchronous game tick. All read and write operations on component data occur during the `tick` method. Accessing this system or the components it manages from other threads will lead to race conditions, `ConcurrentModificationException`, and unpredictable world state corruption.

## API Surface
The public API is primarily for framework integration, not direct developer use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the static query that filters for entities with TransformComponent and NetworkId. |
| tick(dt, systemIndex, store) | void | O(N) | Invoked by the system scheduler each frame. Iterates over all matching entities to update the parent SpatialSystem. |
| getPosition(archetypeChunk, index) | Vector3d | O(1) | Callback for the parent system. Extracts the world position from an entity's TransformComponent. |

## Integration Patterns

### Standard Usage
Developers do not interact with this system directly. Interaction is declarative and managed by the ECS framework. To make an entity managed by this system, simply add the required components to it.

```java
// An entity with these components will be automatically processed
// by NetworkSendableSpatialSystem on the next server tick.
Entity entity = world.createEntity();
entity.addComponent(new TransformComponent(...));
entity.addComponent(new NetworkId(...));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new NetworkSendableSpatialSystem()`. The system is tightly coupled to the world's lifecycle and resource management, which is handled by the server's bootstrap process.
- **Manual Ticking:** Do not call the `tick` method manually. This would bypass the scheduler, disrupt the server's update order, and likely cause severe synchronization issues.
- **Asynchronous Modification:** Do not add or remove a TransformComponent or NetworkId from an entity on a separate thread. All component modifications must be queued and executed on the main server thread to prevent data corruption within the spatial partition.

## Data Pipeline
This system does not operate in a simple linear pipeline. Instead, it reads from the central entity store to populate a shared spatial resource, which is then queried by other systems.

> **Update Flow:**
> ECS Scheduler -> **NetworkSendableSpatialSystem.tick()** -> Iterates matching entities in `EntityStore` -> Calls `getPosition()` -> Parent `SpatialSystem` updates internal `SpatialResource` (e.g., Octree)
>
> **Query Flow (Example):**
> Player Packet System -> Queries `SpatialResource` for entities near player -> Receives list of nearby networkable entities -> Serializes entity data for client

