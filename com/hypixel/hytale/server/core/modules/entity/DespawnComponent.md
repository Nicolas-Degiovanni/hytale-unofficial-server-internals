---
description: Architectural reference for DespawnComponent
---

# DespawnComponent

**Package:** com.hypixel.hytale.server.core.modules.entity
**Type:** Transient

## Definition
```java
// Signature
public class DespawnComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The DespawnComponent is a fundamental data component within the server-side Entity-Component-System (ECS) architecture. It does not contain any logic itself; rather, it acts as a data-holding "tag" that schedules an entity for future removal from the world.

Its primary function is to associate an entity with an absolute point in time for its destruction. This is managed by a dedicated, but separate, "Despawn System" that queries for all entities possessing this component. During each server tick, this system checks if the current server time has surpassed the component's stored `timeToDespawnAt` value. If it has, the system issues a command to destroy the entity.

Using an absolute `java.time.Instant` instead of a relative countdown timer (e.g., a float that is decremented each tick) is a deliberate design choice. This approach ensures that entity despawning is resilient to server lag or fluctuations in tick rate. An entity scheduled to despawn in 5 seconds will be removed at the correct world time, regardless of how many ticks occur in that interval.

The presence of a static `CODEC` field signifies deep integration with the engine's serialization framework. This allows the despawn timer of an entity to be correctly saved during world persistence and restored upon loading.

### Lifecycle & Ownership
- **Creation:** A DespawnComponent is never instantiated directly by a developer. It is created via one of its static factory methods (e.g., `despawnInSeconds`) which require a `TimeResource` to ensure the despawn time is relative to the canonical server clock. The component is then attached to an entity via a `CommandBuffer`, typically using the `trySetDespawn` helper method. This defers the actual attachment to a safe point in the server's tick cycle, preventing concurrency issues.

- **Scope:** The component's lifetime is strictly coupled to the entity it is attached to. It exists only as long as its parent entity exists and the component has not been explicitly removed.

- **Destruction:** The component is destroyed under two conditions:
    1. When its parent entity is destroyed for any reason.
    2. When it is explicitly removed from an entity via a `CommandBuffer` call, which effectively cancels the scheduled despawn.

## Internal State & Concurrency
- **State:** The component's state is mutable, consisting of a single nullable field: `timeToDespawnAt`. This field can be modified after the component has been attached to an entity.

- **Thread Safety:** **This component is not thread-safe.** As with nearly all ECS components, it is designed to be accessed and modified exclusively by the main server thread during the simulation tick. Any attempt to modify an entity's DespawnComponent from an asynchronous task or different thread will result in undefined behavior, including state corruption or server crashes. All modifications must be funneled through a `CommandBuffer`.

## API Surface
The public API is designed around safe, high-level operations rather than direct state manipulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| despawnInSeconds(TimeResource, float) | DespawnComponent | O(1) | **Static Factory.** Creates a new component scheduled to despawn relative to the current server time. |
| despawnInMilliseconds(TimeResource, long) | DespawnComponent | O(1) | **Static Factory.** Creates a new component scheduled to despawn relative to the current server time. |
| trySetDespawn(...) | static void | O(1) | **Primary Integration Point.** Safely adds, updates, or removes a DespawnComponent from an entity using a CommandBuffer. |
| setDespawn(Instant) | void | O(1) | Sets an absolute despawn timestamp. **Warning:** Should only be called from within a system's update logic. |
| getDespawn() | Instant | O(1) | Returns the absolute despawn timestamp, or null if not set. |

## Integration Patterns

### Standard Usage
The correct and safest way to manage an entity's despawn timer is through the static `trySetDespawn` utility method. This method correctly handles all three scenarios: adding a new timer, updating an existing one, or removing it entirely (by passing null for the lifetime).

```java
// How a developer should normally use this
// Assume 'commandBuffer', 'timeResource', and 'entityRef' are available in a system.

// Schedule an entity to despawn in 30 seconds
float lifetimeInSeconds = 30.0f;
DespawnComponent existingComponent = world.getComponent(entityRef, DespawnComponent.getComponentType());

DespawnComponent.trySetDespawn(commandBuffer, timeResource, entityRef, existingComponent, lifetimeInSeconds);

// To cancel a despawn, pass null for the lifetime
DespawnComponent.trySetDespawn(commandBuffer, timeResource, entityRef, existingComponent, null);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new DespawnComponent()`. The static factory methods ensure the component is created relative to the correct server `TimeResource`, not the machine's system clock.

- **Direct Attachment:** Avoid attaching the component directly to an entity outside of the `CommandBuffer` pattern. This bypasses the engine's deferred command system and can lead to concurrent modification exceptions.
    ```java
    // BAD: Bypasses the command buffer, not thread-safe
    entity.putComponent(DespawnComponent.getComponentType(), new DespawnComponent());
    ```

- **Ignoring TimeResource:** Creating a despawn time based on `Instant.now()` will desynchronize the event from the in-game world clock, which can be paused or manipulated. Always use the provided `TimeResource`.

## Data Pipeline
The DespawnComponent does not process data itself; it *is* the data that flows through the entity despawning pipeline.

> Flow:
> Game Logic (e.g., a spell effect expiring) -> `DespawnComponent.trySetDespawn` -> `CommandBuffer` -> **DespawnComponent** (attached to Entity) -> Despawn System (on a future tick) -> Entity Destruction Command -> Entity removed from `EntityStore`

