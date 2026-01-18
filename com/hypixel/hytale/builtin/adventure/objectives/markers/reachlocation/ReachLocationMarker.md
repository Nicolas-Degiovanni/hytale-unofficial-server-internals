---
description: Architectural reference for ReachLocationMarker
---

# ReachLocationMarker

**Package:** com.hypixel.hytale.builtin.adventure.objectives.markers.reachlocation
**Type:** Data Component

## Definition
```java
// Signature
public class ReachLocationMarker implements Component<EntityStore> {
```

## Architecture & Concepts
The ReachLocationMarker is a data component within Hytale's Entity-Component-System (ECS) architecture. It does not contain logic. Instead, it acts as a stateful data tag attached to an entity, designating it as a physical objective location in the game world.

Its primary role is to bridge an in-world entity with a static data definition. The internal *markerId* field serves as a foreign key to a ReachLocationMarkerAsset, which holds descriptive metadata such as the location's display name.

This component is fundamental to the Adventure Mode objective system. Systems that manage quest progression will query for entities with this component to check if a player has physically reached the required coordinates. The internal *players* set tracks the completion state for each player, making it suitable for multiplayer scenarios.

### Lifecycle & Ownership
- **Creation:** ReachLocationMarker instances are managed by the engine's ECS framework. They are typically instantiated in one of two ways:
    1.  **Deserialization:** The static CODEC is used by the EntityStore to construct the component when loading an entity from a world save or prefab. This is the most common creation path.
    2.  **Programmatic:** An objective or scripting system may add a new instance to an entity at runtime via the entity's addComponent method.

- **Scope:** The lifecycle of a ReachLocationMarker is strictly bound to the entity it is attached to. It persists as long as the parent entity exists within the world's EntityStore.

- **Destruction:** The component is destroyed and subsequently garbage collected when its parent entity is removed from the world. There is no manual destruction or cleanup method.

## Internal State & Concurrency
- **State:** This component is **mutable**. Its core state is the `players` set, which is populated at runtime as players interact with the associated entity's trigger volume. The `markerId` is typically considered immutable after creation.

- **Thread Safety:** This component is **not thread-safe**. The internal HashSet is not a concurrent collection. All reads and, more critically, all writes to the `players` set must be synchronized with the main server game loop (tick). Unmanaged access from worker threads, such as networking or physics callbacks, will result in undefined behavior and likely cause a ConcurrentModificationException.

## API Surface
The public API provides access to the component's state and its associated asset data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally registered ComponentType for this class. |
| getMarkerId() | String | O(1) | Returns the unique identifier linking this component to a ReachLocationMarkerAsset. |
| getLocationName() | String | O(log N) | Fetches the display name from the global asset map. Returns null if the asset for markerId is not found. |
| getPlayers() | Set<UUID> | O(1) | Returns a reference to the internal set of player UUIDs that have reached this location. |
| clone() | Component | O(N) | Creates a deep copy of the component, where N is the number of players in the set. |

**WARNING:** The `getPlayers` method returns a direct reference to the internal mutable set, not a copy. Modifying this set externally is the intended pattern for updating state, but it must be done with extreme care regarding concurrency.

## Integration Patterns

### Standard Usage
The component is intended to be retrieved from an entity and modified by a managing system, such as an objective tracker.

```java
// An ObjectiveSystem running on the server tick
void checkPlayerProximity(Player player, Entity objectiveEntity) {
    ReachLocationMarker marker = objectiveEntity.getComponent(ReachLocationMarker.class);

    if (marker != null && isPlayerInTriggerZone(player, objectiveEntity)) {
        // This is the correct way to update state
        marker.getPlayers().add(player.getUUID());
        // ... fire objective completion event
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ReachLocationMarker()`. Components must be added to entities via the `entity.addComponent()` method to ensure they are correctly registered with the EntityStore.

- **Cross-Thread Modification:** Never modify the set returned by `getPlayers` from any thread other than the main server thread. This will corrupt the component's state.

- **State Caching:** Do not retrieve the player set and hold a reference to it across multiple ticks. Always re-fetch the component from the entity on the tick you intend to use it to ensure you have the most recent state.

## Data Pipeline
The ReachLocationMarker is a state container, not an active participant in data transformation. It serves as the terminal point for a gameplay logic pipeline.

> Flow:
> Player Movement Packet -> Server Physics Engine -> Collision/Trigger Event -> ObjectiveSystem -> **ReachLocationMarker** state is updated -> ObjectiveCompletionEvent is fired -> Event Bus -> UI System / Reward System

