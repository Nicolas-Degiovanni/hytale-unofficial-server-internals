---
description: Architectural reference for ObjectiveLocationMarker
---

# ObjectiveLocationMarker

**Package:** com.hypixel.hytale.builtin.adventure.objectives.markers.objectivelocation
**Type:** Component (Data-Oriented)

## Definition
```java
// Signature
public class ObjectiveLocationMarker implements Component<EntityStore> {
```

## Architecture & Concepts

The ObjectiveLocationMarker is a data component within Hytale's Entity Component System (ECS) framework. It does not contain complex logic itself; instead, it serves as a stateful data container that attaches the concept of an "objective trigger area" to an entity in the game world.

This component is the live, in-world instance of a configured `ObjectiveLocationMarkerAsset`. While the asset defines the *template* for a trigger—such as which objective to start, the shape of its activation area, and its trigger conditions—the ObjectiveLocationMarker component represents its physical manifestation and current state within a running game session.

It acts as the bridge between a physical location (represented by its parent entity's position and its own `area` field) and the server's central `ObjectivePlugin`. Systems on the server, such as a hypothetical ObjectiveSystem, will query for entities possessing this component to check for player interaction and manage the lifecycle of adventure mode objectives.

## Lifecycle & Ownership

-   **Creation:** An ObjectiveLocationMarker is never instantiated directly. It is created by the server's ECS framework when an entity is loaded from world storage or spawned dynamically. The static `CODEC` field is used by the engine to deserialize entity data from a saved format into a live component instance.
-   **Scope:** The lifecycle of this component is strictly bound to the lifecycle of its parent entity. It exists only as long as the entity it is attached to exists in the `EntityStore`.
-   **Destruction:** The component is marked for garbage collection when its parent entity is unloaded or destroyed. Any active objectives associated with it must be explicitly canceled by a managing system (e.g., via `ObjectivePlugin.cancelObjective`) prior to entity destruction to prevent orphaned state.

## Internal State & Concurrency

-   **State:** The component is highly **mutable**. Its primary purpose is to hold the runtime state of an objective trigger, including the `activeObjectiveUUID` and the dynamically calculated trigger `area` (which may be rotated based on the parent entity's orientation). The `updateLocationMarkerValues` method is the main entry point for state changes, synchronizing the component with its backing asset.
-   **Thread Safety:** This component is **not thread-safe** and must be considered thread-hostile. As with all ECS components, it is designed to be exclusively accessed and modified by the main server thread during the game tick. Unsynchronized access from other threads will result in state corruption, race conditions, and server instability.

**WARNING:** Any system interacting with an ObjectiveLocationMarker must ensure its operations are scheduled on the main game loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique, registered type identifier for this component. |
| updateLocationMarkerValues(...) | void | O(N) | Synchronizes the component's state from an asset. May trigger side effects, such as canceling the currently active objective if the new configuration specifies a different one. |
| getActiveObjective() | Objective | O(1) | Returns a cached reference to the live Objective instance. This can be null if no objective is active. |
| clone() | Component | O(1) | Creates a shallow copy of the component. Internal object references like `area` and `activeObjective` are copied, not deep-cloned. |

## Integration Patterns

### Standard Usage

This component is primarily managed by server-side systems. A developer's interaction is typically limited to defining the `ObjectiveLocationMarkerAsset` that configures its behavior. A system would retrieve and interact with the component as follows.

```java
// A hypothetical system processing an entity with this component
Entity entity = ...;
Store<EntityStore> store = ...; // The entity store

ObjectiveLocationMarker marker = entity.get(ObjectiveLocationMarker.getComponentType());
if (marker != null) {
    // Check if a player is inside the marker's area
    if (isPlayerInArea(player, marker.getArea())) {
        // Further logic to check trigger conditions and start the objective
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ObjectiveLocationMarker()`. Components must be created and managed via an `EntityManager` or `EntityStore` to ensure they are correctly registered within the ECS framework.
-   **State Caching:** Do not cache the reference returned by `getActiveObjective()` across multiple game ticks. The active objective can be changed or canceled by other systems at any time, leading to a stale and invalid reference. Re-fetch the component from the entity each tick.
-   **Manual Serialization:** Do not attempt to serialize or deserialize this component manually. Rely exclusively on the provided static `CODEC` to ensure data integrity and forward compatibility.

## Data Pipeline

The flow of data from configuration to a live, interactive world object follows a clear path.

> Flow:
> 1. **ObjectiveLocationMarkerAsset** (Defined in a JSON or HOCON file)
> 2. Loaded by the server's **AssetManager**
> 3. Deserialized via **ObjectiveLocationMarker.CODEC** during entity creation
> 4. Instance attached to an **Entity** in the **EntityStore**
> 5. **ObjectiveSystem** queries for the component and evaluates its state (e.g., player enters `area`)
> 6. System calls **ObjectivePlugin** to start a new objective
> 7. The `activeObjectiveUUID` and `activeObjective` fields are populated, reflecting the new world state.

