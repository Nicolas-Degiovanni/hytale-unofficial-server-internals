---
description: Architectural reference for PatrolPathMarkerEntity
---

# PatrolPathMarkerEntity

**Package:** com.hypixel.hytale.builtin.path.entities
**Type:** Managed Entity

## Definition
```java
// Signature
public class PatrolPathMarkerEntity extends Entity implements IPrefabPathWaypoint {
```

## Architecture & Concepts

The PatrolPathMarkerEntity is a server-side entity that represents a single, physical waypoint within a logical patrol path. It is a fundamental building block for defining NPC movement and behavior sequences in the world.

Architecturally, this class bridges the gap between the abstract concept of a path (an ordered list of points) and its concrete implementation within the game world. While a logical path, represented by an IPrefabPath implementation like PatrolPath, exists in memory, each PatrolPathMarkerEntity is a full-fledged entity managed by the world's EntityStore. This means each waypoint has a persistent UUID, a TransformComponent for position and rotation, and a defined lifecycle.

These entities are typically invisible to players in non-creative game modes. This positions them as a level design and world-building tool, used to choreograph entity movements without cluttering the game view for standard players.

The serialization of this entity is handled by a static BuilderCodec instance. This codec is responsible for reading and writing the entity's state to and from the world's storage, ensuring that patrol paths persist across server sessions. It includes versioning to maintain compatibility with older world data.

## Lifecycle & Ownership

-   **Creation:** A PatrolPathMarkerEntity is instantiated by the server engine, typically as a result of a world generation process, a prefab being placed, or direct creation via in-game editing tools. The default constructor creates the object, but it remains in a latent state until the `initialise` method is called. This second-stage initialization is critical, as it registers the waypoint with the central WorldPathData resource, linking it to its logical parent path.

-   **Scope:** The entity's lifetime is tied to its presence in the World. It persists as long as it is registered in the world's EntityStore. Its state is saved to disk as part of the world chunk it occupies.

-   **Destruction:** The entity is destroyed when it is removed from the world, for example, when a player in creative mode breaks it. The `onReplaced` method is the primary cleanup hook, which nullifies its path association and triggers its removal from the game world.

## Internal State & Concurrency

-   **State:** The PatrolPathMarkerEntity is a mutable, stateful object. It stores its own configuration, such as its order in the sequence, the pause time for an NPC at this point, and a target observation angle. Crucially, it holds a direct reference (`parentPath`) to the logical path object it belongs to, which is populated during initialization.

-   **Thread Safety:** This class is **not thread-safe**. As a subclass of Entity, all modifications to its state and all method calls must be performed on the main server thread that owns the entity's world instance. Methods that modify state, such as `setOrder` or `setPauseTime`, call `markNeedsSave` to queue the entity for persistence at the end of the current server tick. Unsynchronized access from other threads will lead to race conditions and world corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| initialise(...) | void | O(N) | Second-stage constructor. Links the entity to a logical path, registers it, and sets its core properties. Complexity depends on path registration logic. |
| getWaypointPosition(...) | Vector3d | O(1) | Retrieves the world-space position of the waypoint from its associated TransformComponent. |
| getWaypointRotation(...) | Vector3f | O(1) | Retrieves the world-space rotation of the waypoint from its associated TransformComponent. |
| setOrder(int order) | void | O(1) | Sets the waypoint's position in the path sequence and marks the entity for saving. **Warning:** This does not automatically re-sort the parent path. |
| onReplaced() | void | O(1) | Lifecycle hook called before destruction. Deregisters the entity from its parent path. |
| isHiddenFromLivingEntity(...) | boolean | O(1) | Determines visibility. Returns true if the target entity is not a player in Creative mode. |

## Integration Patterns

### Standard Usage

A PatrolPathMarkerEntity should never be managed directly. It should be created and configured by higher-level systems, such as world generation or editor tools, which have access to the server's ComponentAccessor. The `initialise` method is the correct entry point for making the entity functional after instantiation.

```java
// Example: Spawning a new waypoint within a server-side system
ComponentAccessor<EntityStore> accessor = world.getComponentAccessor();
WorldPathData pathData = accessor.getResource(WorldPathData.getResourceType());

// 1. Create the entity in the world
PatrolPathMarkerEntity marker = world.createEntity(PatrolPathMarkerEntity.class, position);

// 2. Initialize its path properties
UUID targetPathId = ...;
String targetPathName = "Zone1_GuardPatrol";
int worldGenId = 0; // Or a valid ID for prefab paths

marker.initialise(targetPathId, targetPathName, -1, 5.0, 90.0f, worldGenId, accessor);

// The marker is now live and part of the "Zone1_GuardPatrol" path.
// The -1 index tells it to append itself to the end of the path.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation without Initialization:** Creating an instance with `new PatrolPathMarkerEntity()` without subsequently adding it to a world and calling `initialise` will result in a non-functional, orphaned object that is not part of any pathing system.

-   **State Manipulation Outside Server Tick:** Modifying the entity's properties from an asynchronous task or a different thread is strictly forbidden and will cause unpredictable behavior and data corruption.

-   **Manual TransformComponent Access:** While possible, developers should prefer using `getWaypointPosition` and `getWaypointRotation` over fetching the TransformComponent manually, as this respects the interface contract defined by IPrefabPathWaypoint.

## Data Pipeline

The PatrolPathMarkerEntity serves as a data point in two primary flows: world persistence and AI consumption.

> **Persistence Flow (World Load):**
> World Save File (Binary Data) -> Server Engine -> **PatrolPathMarkerEntity.CODEC** -> Deserializes fields (pathId, order, etc.) -> `PatrolPathMarkerEntity` instance created -> `initialise` is called -> Entity registers with `WorldPathData`.

> **AI Consumption Flow (Runtime):**
> NPC AI System -> Requests path from `WorldPathData` -> Receives `PatrolPath` object -> Iterates waypoints -> Accesses **PatrolPathMarkerEntity** instance -> Calls `getWaypointPosition()` -> AI Pathfinding system receives target coordinate.

