---
description: Architectural reference for ItemPhysicsComponent
---

# ItemPhysicsComponent

**Package:** com.hypixel.hytale.server.core.modules.entity.item
**Type:** Data Component

## Definition
```java
// Signature
@Deprecated
public class ItemPhysicsComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The ItemPhysicsComponent is a server-side, data-only component within the Hytale Entity-Component-System (ECS) architecture. Its sole purpose is to store transient, physics-related state for a dropped item entity. It acts as a simple data container, holding the entity's velocity and the result of its most recent collision detection query.

**WARNING:** This component is marked as **Deprecated**. Its use is strongly discouraged in new development. It likely exists for legacy purposes and has been superseded by a more generic or performant physics component. Production systems should migrate away from this implementation to avoid building on an obsolete pattern.

This component is designed to be manipulated exclusively by a dedicated physics system. The system reads the state, performs physics calculations (e.g., applying gravity, checking for collisions), and writes the new state back into the component's public fields for the next game tick.

### Lifecycle & Ownership
- **Creation:** An ItemPhysicsComponent is instantiated and attached to an entity when that entity is created, typically by an entity factory or spawner. The `clone` method also allows for its creation when an entity is duplicated.
- **Scope:** The lifecycle of this component is strictly bound to the lifecycle of the entity it is attached to. It persists only as long as the item entity exists in the world.
- **Destruction:** The component is marked for garbage collection and destroyed when its parent entity is removed from the `EntityStore`, for example, when an item is picked up by a player or despawns.

## Internal State & Concurrency
- **State:** This component is a highly mutable data bag. Its public fields, `scaledVelocity` and `collisionResult`, are intended for direct, high-frequency modification by the server's physics processing loop. It holds no logic, only state.

- **Thread Safety:** This component is **not thread-safe**. It is designed to be accessed and modified within a single, well-defined thread of execution, such as the main server tick loop or a dedicated physics thread. Unsynchronized access from multiple threads will lead to race conditions, physics glitches, and world corruption.

## API Surface
The primary interface for this component is direct field access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component, used for registration and lookup within the ECS framework. |
| clone() | Component | O(1) | Creates a new instance of the component. **WARNING:** This performs a shallow copy of the internal `Vector3d` and `CollisionResult` objects, not a deep copy. |

## Integration Patterns

### Standard Usage
This component is not meant to be used directly by gameplay logic. Instead, a physics system retrieves it from an entity to process movement.

```java
// Example within a hypothetical PhysicsSystem
void processItemMovement(Entity itemEntity) {
    ItemPhysicsComponent physics = itemEntity.getComponent(ItemPhysicsComponent.class);
    if (physics == null) {
        return;
    }

    // 1. Read current state
    Vector3d currentVelocity = physics.scaledVelocity;

    // 2. Apply physics logic (e.g., gravity, drag)
    Vector3d newVelocity = applyForces(currentVelocity);

    // 3. Perform collision checks and write result
    CollisionResult result = world.traceRay(itemEntity.getPosition(), newVelocity);
    physics.collisionResult = result;

    // 4. Update velocity for the next tick
    physics.scaledVelocity = newVelocity;
}
```

### Anti-Patterns (Do NOT do this)
- **Usage of Deprecated Class:** The most significant anti-pattern is using this component at all. It is deprecated and should be replaced by the modern, supported equivalent.
- **Direct Instantiation:** Do not use `new ItemPhysicsComponent()`. Components must be added to entities via the appropriate entity manager or world API (e.g., `entity.addComponent(...)`) to ensure they are correctly registered with the ECS.
- **Concurrent Modification:** Never read or write to the component's fields from multiple threads without external locking. All physics interactions for an entity must be serialized.
- **Misunderstanding `clone`:** Do not assume `clone()` creates a fully independent copy. Modifying the `scaledVelocity` of a cloned component may affect the original if the Vector3d object reference was shared.

## Data Pipeline
This component serves as a stateful node in the server's physics pipeline for item entities.

> Flow:
> **Physics System (Read)** -> Reads `scaledVelocity` -> **Collision Engine** -> Calculates collision and writes to `collisionResult` -> **Physics System (Write)** -> Updates `scaledVelocity` -> Next Game Tick

