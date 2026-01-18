---
description: Architectural reference for HeadRotation
---

# HeadRotation

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Transient

## Definition
```java
// Signature
public class HeadRotation implements Component<EntityStore> {
```

## Architecture & Concepts
The HeadRotation class is a fundamental data component within the server-side Entity-Component-System (ECS) architecture. Its sole responsibility is to store the orientation of an entity's head, represented by a three-dimensional vector for pitch, yaw, and roll.

As a component, it is a plain data container with no inherent logic. It is designed to be attached to an Entity and manipulated by various Systems, such as an AI system determining where a mob should look, or a PlayerControl system processing network input for camera movement.

The presence of a static **CODEC** field indicates that this component is a critical part of the data serialization pipeline. The engine uses this codec to efficiently encode and decode HeadRotation state for network synchronization with clients and for persisting entity state to the world save (EntityStore). Its registration via `EntityModule.get().getHeadRotationComponentType()` makes it a known, manageable type within the server's core entity framework.

## Lifecycle & Ownership
- **Creation:** A HeadRotation component is instantiated and attached to an Entity when that entity is first created, provided its archetype requires head orientation (e.g., players, mobs, but not a static chest). It is also created during deserialization when an entity is loaded from storage or spawned from a network packet.
- **Scope:** The lifecycle of a HeadRotation instance is strictly bound to the lifecycle of its parent Entity. It persists only as long as the entity exists in the world.
- **Destruction:** The component is marked for garbage collection when its parent Entity is destroyed and removed from the EntityStore. There is no manual destruction method; ownership is managed entirely by the ECS framework.

**WARNING:** Caching or holding a direct reference to a HeadRotation component outside the scope of a single system update is a severe anti-pattern. The component may be destroyed at any time along with its parent entity, leading to stale references and memory leaks.

## Internal State & Concurrency
- **State:** The component's state is **Mutable**. It encapsulates a single `Vector3f` object. While the reference to this vector is final, the vector's internal pitch, yaw, and roll values are modified by methods like `setRotation` and `teleportRotation`.
- **Thread Safety:** This component is **not thread-safe**. It contains no internal locking mechanisms. All reads and writes to a HeadRotation instance must be performed by systems operating on the main server thread to prevent race conditions and data corruption. External synchronization is required if off-thread access is unavoidable, though such a pattern is strongly discouraged.

## API Surface
The public API provides methods for state manipulation and orientation-based calculations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setRotation(Vector3f) | void | O(1) | Overwrites the current rotation with the provided vector's values. |
| getRotation() | Vector3f | O(1) | Returns a direct reference to the internal rotation vector. |
| getDirection() | Vector3d | O(1) | Calculates and returns a new normalized direction vector based on pitch and yaw. |
| getAxisDirection() | Vector3i | O(1) | Calculates the dominant cardinal direction (e.g., {1,0,0}) as an integer vector. |
| teleportRotation(Vector3f) | void | O(1) | A specialized setter that updates rotation components individually, safely ignoring NaN inputs. Primarily used for applying partial state updates from network packets. |
| clone() | HeadRotation | O(1) | Creates a new HeadRotation instance with a deep copy of the internal rotation vector. |

## Integration Patterns

### Standard Usage
A System retrieves the component from an entity to read or update its state within a single tick. This is the only correct way to interact with the component.

```java
// Example from within a hypothetical AI system
void updateLookDirection(Entity entity, Vector3f target) {
    HeadRotation rotation = entity.getComponent(HeadRotation.class);
    if (rotation != null) {
        // Logic to calculate new pitch/yaw towards target
        rotation.setRotation(target);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new HeadRotation()`. Components must be added to entities via the ECS framework, for example `entity.addComponent(new HeadRotation())`, to ensure proper registration and lifecycle management.
- **Concurrent Modification:** Do not read or write to this component from an asynchronous task or a different thread without explicit, engine-provided synchronization mechanisms. The state is not guaranteed to be consistent.
- **Stale References:** Do not store a reference to a HeadRotation component in a long-lived object. Always re-fetch the component from the entity on each update tick.

## Data Pipeline
HeadRotation is a key node in the entity state synchronization pipeline.

> **Flow (Input):**
> Client Input Packet -> Server Network Layer -> Player Control System -> **HeadRotation.setRotation()**

> **Flow (Output):**
> Entity State Snapshot System -> **HeadRotation.getRotation()** -> Serializer (using CODEC) -> Server Network Layer -> Client Render Thread

