---
description: Architectural reference for PickupItemComponent
---

# PickupItemComponent

**Package:** com.hypixel.hytale.server.core.modules.entity.item
**Type:** Transient Data Component

## Definition
```java
// Signature
public class PickupItemComponent implements Component<EntityStore> {
```

## Architecture & Concepts

The PickupItemComponent is a fundamental data structure within Hytale's Entity-Component-System (ECS) architecture. It does not contain any logic itself; instead, it serves as a stateful data tag that signals to other systems that an item entity is in the process of being picked up by a target entity.

When this component is attached to an item entity, it effectively initiates a short-lived state machine. An external system, likely an `ItemPickupSystem`, is responsible for observing entities with this component. Each game tick, the system reads the component's state—specifically the `targetRef` and remaining `lifeTime`—to interpolate the item's physical position, creating a smooth animation of it traveling towards the target.

The presence of a static `CODEC` field is a critical architectural detail. It indicates that the component's state is fully serializable. This allows the server to synchronize the pickup animation across all clients by sending the component's data when it is created. It also implies potential for persistence in world saves.

This component acts as the data-driven trigger for one of the most common interactions in the game world: acquiring items.

## Lifecycle & Ownership

-   **Creation:** A PickupItemComponent is not instantiated directly by general-purpose code. It is created and attached to an item entity by a higher-level game logic system. This typically occurs when a player or other entity meets the criteria to collect an item, such as proximity or direct interaction.
-   **Scope:** The component's lifetime is extremely brief and is explicitly managed by its internal timer. It persists only for the duration of the pickup animation, which defaults to 0.15 seconds. Its scope is bound to the specific item entity it is attached to.
-   **Destruction:** The component is marked for removal once its `lifeTime` is less than or equal to zero. The same system responsible for processing the animation is also responsible for removing the component from the entity. Upon removal, the system will typically finalize the item transfer to the target's inventory and destroy the item entity from the world.

## Internal State & Concurrency

-   **State:** The component's state is highly mutable. The `lifeTime` and `finished` fields are designed to be modified on every server tick by the authoritative processing system. The `targetRef` and `startPosition` are immutable after creation, defining the fixed parameters of the pickup action.
-   **Thread Safety:** **This component is not thread-safe.** Like most ECS components, it is designed to be accessed and mutated exclusively by a single thread within the main server game loop. Unsynchronized access from other threads will lead to race conditions, state corruption, and unpredictable behavior. All interactions must be marshaled through the main world update thread.

## API Surface

The public API is minimal, exposing only the necessary state for an external system to manage the pickup process.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decreaseLifetime(float amount) | void | O(1) | The primary mutation method. Called every tick by the processing system to advance the animation timer. |
| hasFinished() | boolean | O(1) | State check used by systems to determine if the pickup process is complete. |
| getTargetRef() | Ref<EntityStore> | O(1) | Returns a reference to the entity that will receive the item. |
| getStartPosition() | Vector3d | O(1) | Returns the world position where the item began its pickup travel. |
| clone() | PickupItemComponent | O(1) | Creates a deep copy of the component's state. |

## Integration Patterns

### Standard Usage

This component should only be accessed and managed within a dedicated ECS system. The system queries for entities possessing this component and updates their state and position each tick.

```java
// Example logic within an ItemPickupSystem's update method
for (Entity itemEntity : world.getEntitiesWith(PickupItemComponent.class)) {
    PickupItemComponent pickup = itemEntity.getComponent(PickupItemComponent.class);

    if (pickup.hasFinished()) {
        continue;
    }

    pickup.decreaseLifetime(deltaTime);

    if (pickup.getLifeTime() <= 0.0f) {
        pickup.setFinished(true);
        // Finalize item transfer to pickup.getTargetRef()
        // Remove component and destroy itemEntity
    } else {
        // Calculate new position based on interpolation and update entity transform
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create this component with `new PickupItemComponent()`. Components must be added to entities via the appropriate `EntityManager` or `Entity` methods to ensure they are correctly registered with the world systems.
-   **External State Mutation:** Do not manually call `setFinished(true)` or arbitrarily modify the `lifeTime` from outside the authoritative processing system. Doing so will break the animation and can lead to items being lost or duplicated.
-   **Long-Lived References:** Do not hold a reference to this component outside the scope of a single system update. Its lifecycle is extremely short, and a stored reference will quickly become stale, pointing to a component that has been removed from its entity.

## Data Pipeline

The component acts as a transient data packet within the server's core loop, driving a specific game mechanic from initiation to completion.

> Flow:
> Proximity System detects Player near Item -> System attaches **PickupItemComponent** to Item Entity -> ItemPickupSystem reads component state each tick -> Item Entity's Transform is updated -> When `lifeTime` expires, System removes component and transfers item data -> Item Entity is destroyed.

