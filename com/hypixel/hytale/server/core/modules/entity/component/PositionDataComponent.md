---
description: Architectural reference for PositionDataComponent
---

# PositionDataComponent

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Data Component

## Definition
```java
// Signature
public class PositionDataComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The PositionDataComponent is a server-side, data-only component within the Entity-Component-System (ECS) architecture. It serves as a high-frequency cache for an entity's immediate environmental context, specifically storing the block types the entity is currently inside and standing upon.

This component's primary architectural function is to decouple complex world-state queries from gameplay logic. Instead of every system (AI, audio, effects) independently querying the world grid to determine an entity's footing, a single, authoritative system (e.g., a PhysicsSystem) performs this calculation once per tick and stores the result in this component. Downstream systems can then read this cached state with near-zero performance cost.

It is a fundamental building block for creating responsive and emergent behaviors based on an entity's interaction with the game world.

### Lifecycle & Ownership
- **Creation:** A PositionDataComponent is instantiated and attached to an Entity when that Entity is created or loaded into the world. This is typically managed by the EntityModule or a spawner system. The clone method facilitates entity duplication and templating.
- **Scope:** The component's lifecycle is strictly bound to its parent Entity. It persists as long as the Entity exists in the world.
- **Destruction:** The component is marked for garbage collection when its parent Entity is destroyed or when the component is explicitly removed from the Entity via the EntityManager.

## Internal State & Concurrency
- **State:** The internal state is mutable and consists of two integer identifiers. It is designed to be updated every server tick for any mobile entity, reflecting its current position relative to the world grid.

- **Thread Safety:** This component is **not thread-safe** and is designed for single-threaded access within the main server game loop. All reads and writes must be synchronized with the server tick.

    **WARNING:** Modifying this component from an asynchronous task, worker thread, or network thread without explicit locking will lead to race conditions, world desynchronization, and undefined behavior. Systems should read the state at the beginning of their update cycle and assume it is constant for the duration of that tick.

## API Surface
The public contract is minimal, focusing exclusively on state management.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the registered type definition for this component from the EntityModule. |
| clone() | Component | O(1) | Creates a deep copy of the component with identical state. |
| getInsideBlockTypeId() | int | O(1) | Returns the cached ID of the block the entity's bounding box occupies. |
| setInsideBlockTypeId(int) | void | O(1) | Sets the cached ID of the block the entity is inside. |
| getStandingOnBlockTypeId() | int | O(1) | Returns the cached ID of the block immediately below the entity. |
| setStandingOnBlockTypeId(int) | void | O(1) | Sets the cached ID of the block the entity is standing on. |

## Integration Patterns

### Standard Usage
A system responsible for physics or entity movement is expected to update this component each tick. Other systems then consume this data to drive gameplay logic.

```java
// Inside a hypothetical PhysicsSystem update method
void processEntity(Entity entity) {
    // Assume TransformComponent and WorldQueryService exist
    TransformComponent transform = entity.getComponent(TransformComponent.class);
    PositionDataComponent posData = entity.getComponent(PositionDataComponent.class);

    if (transform == null || posData == null) {
        return; // Entity does not have the required components
    }

    // Perform world queries based on the entity's new position
    int insideBlock = world.getBlockAt(transform.getPosition());
    int groundBlock = world.getBlockAt(transform.getPosition().add(0, -1, 0));

    // Update the component's state for other systems to read
    posData.setInsideBlockTypeId(insideBlock);
    posData.setStandingOnBlockTypeId(groundBlock);
}
```

### Anti-Patterns (Do NOT do this)
- **External Instantiation:** Do not use `new PositionDataComponent()`. Components must be added to an entity through the appropriate entity manager or world context (e.g., `entity.addComponent(...)`) to ensure they are correctly registered and tracked.
- **Stale References:** Do not cache a reference to a PositionDataComponent across multiple ticks. The parent entity could be destroyed, leading to a null reference or use-after-free bugs. Always re-fetch the component from the entity each tick.
- **Asynchronous Modification:** Never modify the component's state from a separate thread. This will break the determinism of the server tick and cause unpredictable physics and AI behavior.

## Data Pipeline
This component acts as a staging area for data flowing from the world simulation to various gameplay systems.

> Flow:
> Physics System calculates new position -> World Grid Query -> **PositionDataComponent** (State Updated) -> AI System / Audio System / Effects System (State Read)

