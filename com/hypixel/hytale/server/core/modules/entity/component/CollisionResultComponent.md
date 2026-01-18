---
description: Architectural reference for CollisionResultComponent
---

# CollisionResultComponent

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Transient Data Component

## Definition
```java
// Signature
public class CollisionResultComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The CollisionResultComponent is a data-only component within the server-side Entity-Component-System (ECS) architecture. It serves as a transient data container that holds the state and results of a physics collision query for a single entity.

Its primary architectural function is to decouple the *intent* to move an entity from the *execution* of the physics simulation. A high-level system (like a MovementSystem) populates this component to signal a desired state change. A low-level system (the PhysicsSystem) then processes it, performs the collision check, and stores the outcome back into the component. This allows for a clean separation of concerns and organizes the game loop into distinct phases: intent, simulation, and resolution.

The component's state is managed by a simple flag, *pendingCollisionCheck*, which acts as a signal to the physics processing stage of the main server tick, indicating that this entity requires a collision calculation.

## Lifecycle & Ownership
- **Creation:** This component is not persistent. It is typically added to an Entity on-demand by a system responsible for entity movement (e.g., a PlayerMovementSystem or an AiNavigationSystem) when that entity attempts to change its position within the world.
- **Scope:** The lifetime of a CollisionResultComponent's data is extremely short, usually confined to a single server tick. It is populated at the beginning of a movement calculation, processed by the physics engine during the simulation phase, and its results are consumed before the tick ends.
- **Destruction:** The component may be removed from the entity after its results are processed, or it may be kept and reset via *resetLocationChange* for performance reasons, avoiding repeated allocation and deallocation for entities that move frequently. It is ultimately garbage collected when the parent entity is destroyed.

## Internal State & Concurrency
- **State:** The component's state is highly mutable and represents a snapshot of a single physics query. It does not cache data across multiple ticks. The internal Vector3d and CollisionResult objects are mutable.

- **Thread Safety:** **Not thread-safe.** This component is designed for single-threaded access within the main server game loop. All reads and writes must be synchronized with the server tick. Unmanaged access from worker threads, network threads, or asynchronous tasks will lead to race conditions, physics glitches, and world state corruption.

    **WARNING:** The *clone* method performs a **shallow copy**. The internal Vector3d and CollisionResult objects are shared between the original and the clone. Modifying a vector from a cloned instance will also modify the vector in the original. This behavior is for performance and assumes the clone is for temporary, read-only inspection.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally unique type identifier for this component from the EntityModule. |
| markPendingCollisionCheck() | void | O(1) | Flags the entity for processing by the collision system in the current tick. |
| consumePendingCollisionCheck() | void | O(1) | Clears the processing flag. This is called by the physics system after the check is complete. |
| resetLocationChange() | void | O(1) | Resets the movement offset and pending flag, preparing the component for the next tick. |
| clone() | Component | O(1) | Creates a shallow copy of this component. See warning regarding thread safety. |

## Integration Patterns

### Standard Usage
The component is used as a communication channel between movement logic and the core physics engine within a single tick.

```java
// In a MovementSystem processing an entity
CollisionResultComponent collision = entity.getOrAddComponent(CollisionResultComponent.class);

// Set the proposed movement
collision.getCollisionStartPosition().assign(entity.getPosition());
collision.getCollisionPositionOffset().assign(desiredVelocity.mul(deltaTime));

// Signal the physics engine to process this entity
collision.markPendingCollisionCheck();

// ... later in the tick, after the PhysicsSystem has run ...

CollisionResult result = collision.getCollisionResult();
if (result.hasCollided()) {
    // Adjust final position based on collision
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use *new CollisionResultComponent()*. The component must be managed by the entity it is attached to. Always use *entity.getOrAddComponent* or a similar entity framework method.
- **State Retention:** Do not read data from this component from a previous tick. Its state is volatile and is not guaranteed to be valid or relevant after the tick in which it was processed.
- **Asynchronous Modification:** Do not modify this component from an asynchronous callback or a separate thread. All state changes must originate from the main server thread to prevent desynchronization of the physics world.

## Data Pipeline
The component acts as a data payload that flows through different systems during the server's update loop.

> Flow:
> Movement Intent (e.g., AI, Player Input) -> `MovementSystem` -> **CollisionResultComponent** (Populated with offset, marked as pending) -> `PhysicsSystem` (Reads component, performs world query, updates result field) -> `MovementSystem` (Reads result, updates final Entity PositionComponent)

