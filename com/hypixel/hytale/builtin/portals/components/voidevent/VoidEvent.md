---
description: Architectural reference for VoidEvent
---

# VoidEvent

**Package:** com.hypixel.hytale.builtin.portals.components.voidevent
**Type:** Transient Data Component

## Definition
```java
// Signature
public class VoidEvent implements Component<EntityStore> {
```

## Architecture & Concepts
The VoidEvent component is a state container within Hytale's Entity Component System (ECS). It does not contain any logic itself; rather, it holds all the necessary data for a controlling system (e.g., a hypothetical VoidEventSystem) to manage a world event.

Its primary architectural purpose is to track the state and spatial distribution of "void spawners" associated with a single, ongoing event. The core of its design is the **SpatialHashGrid**, a performance-critical data structure used to enforce gameplay rules, specifically the minimum distance between active spawners. By partitioning the world space, it allows for O(1) average-case complexity when querying for nearby spawners, preventing clustering and ensuring a balanced distribution.

The component also maintains a reference to the event's current phase, the **VoidEventStage**. This allows the controlling system to alter its behavior over time, progressing the event through a predefined sequence of states.

## Lifecycle & Ownership
- **Creation:** A VoidEvent component is instantiated and attached to an EntityStore by a higher-level game system when a void event is triggered in the world. It is never intended to be created directly by user code.
- **Scope:** The component's lifetime is strictly bound to the entity to which it is attached. It persists as long as the entity exists and the component is not explicitly removed by a system.
- **Destruction:** The component is marked for garbage collection when its parent entity is destroyed or when a system removes it, signifying the end of the event.

## Internal State & Concurrency
- **State:** This component is highly mutable. Its internal fields, `voidSpawners` and `activeStage`, are designed to be frequently updated by its managing system throughout the event's duration. The `voidSpawners` grid acts as a live cache of spawner entity locations.
- **Thread Safety:** **This component is not thread-safe.** All internal collections are non-synchronized. Any and all modifications to the component's state, including adding or removing spawners from the `SpatialHashGrid`, must be performed exclusively on the main world thread to prevent catastrophic race conditions and data corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component from the plugin registry. |
| getConfig(World) | VoidEventConfig | O(1) | Fetches the world-specific configuration for the event from the active gameplay configuration. |
| getVoidSpawners() | SpatialHashGrid | O(1) | Returns a direct reference to the grid managing spawner locations. |
| getActiveStage() | VoidEventStage | O(1) | Returns the current phase of the event, or null if no stage is active. |
| setActiveStage(stage) | void | O(1) | Sets the current phase of the event. |
| clone() | Component | O(1) | **WARNING:** Performs a shallow copy. The new component shares references to the original `SpatialHashGrid` and `activeStage`. See Anti-Patterns. |

## Integration Patterns

### Standard Usage
A managing system queries for an entity with the VoidEvent component, reads its state, and executes logic accordingly.

```java
// Example from within a hypothetical VoidEventSystem
void processEvent(EntityStore entity) {
    VoidEvent event = entity.getComponent(VoidEvent.getComponentType());
    if (event == null) {
        return;
    }

    VoidEventStage currentStage = event.getActiveStage();
    if (currentStage != null && shouldSpawn(currentStage)) {
        SpatialHashGrid<Ref<EntityStore>> spawners = event.getVoidSpawners();
        // ... logic to find a valid spawn location using the grid
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new VoidEvent()`. Components must be created and attached to entities via the `EntityStore.addComponent` method to be correctly registered with the ECS.
- **State Sharing via Clone:** The `clone` method is extremely hazardous. It creates a new component instance that points to the *exact same* `SpatialHashGrid` and `activeStage` objects. Modifying the cloned component's state will corrupt the state of the original, leading to unpredictable behavior and bugs that are difficult to trace.
- **Cross-Thread Modification:** Accessing or modifying the `voidSpawners` grid from any thread other than the one that owns the parent entity's world tick will break thread-safety guarantees and corrupt the event state.

## Data Pipeline
The VoidEvent component is a data sink and source, not a processing node. It is acted upon by a system as part of the main game loop.

> Flow:
> VoidEventSystem (Tick) -> Reads **VoidEvent** state -> Modifies **VoidEvent** state (e.g., adds spawner to grid) -> World (Entity Spawning)

