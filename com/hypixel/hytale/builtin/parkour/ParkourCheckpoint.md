---
description: Architectural reference for ParkourCheckpoint
---

# ParkourCheckpoint

**Package:** com.hypixel.hytale.builtin.parkour
**Type:** Data Component

## Definition
```java
// Signature
public class ParkourCheckpoint implements Component<EntityStore> {
```

## Architecture & Concepts
The ParkourCheckpoint is a data-only component within Hytale's Entity Component System (ECS). Its sole purpose is to tag an entity, marking it as a sequential checkpoint within a parkour course. It does not contain any game logic itself; instead, it serves as a data-bearing flag for other systems, such as a hypothetical ParkourSystem, to query and act upon.

The class is fundamentally integrated with the engine's serialization and world-state management through two key features:
1.  **Component Interface:** By implementing `Component<EntityStore>`, it signals to the engine that it is a piece of state managed by the server's primary world container, the EntityStore.
2.  **Static CODEC:** The public static `CODEC` field is a critical integration point. It provides the engine with the complete blueprint for serializing an instance to disk or network streams and deserializing it back into a live object. This allows parkour courses to be saved in world data and replicated to clients.

This component is a classic example of ECS design, separating data (the checkpoint's existence and index) from behavior (the logic that handles what happens when a player reaches a checkpoint).

### Lifecycle & Ownership
-   **Creation:** An instance of ParkourCheckpoint is created under two circumstances:
    1.  **Programmatically:** By a server-side system or game script when a parkour course is procedurally generated or modified. The component is then attached to a target entity.
    2.  **Deserialization:** By the engine's `CODEC` system when a world chunk containing an entity with this component is loaded from disk or received over the network.
-   **Scope:** The lifecycle of a ParkourCheckpoint is strictly bound to the lifecycle of the entity to which it is attached. It persists as long as the host entity exists within the EntityStore.
-   **Destruction:** The component is marked for garbage collection when its host entity is removed from the world. There is no manual destruction method; its memory is managed by the ECS framework and the JVM.

## Internal State & Concurrency
-   **State:** The component holds a single piece of mutable state: the integer `index`. This value is intended to be set at creation and remain constant thereafter. It does not cache any external data.
-   **Thread Safety:** **This class is not thread-safe.** As a plain data object, it possesses no internal locking or synchronization. All access and modification must be performed on the main server thread responsible for the world tick. Unsynchronized access from other threads will result in data corruption and undefined behavior.

## API Surface
The public API is minimal, reflecting its role as a simple data container.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component from the ParkourPlugin. |
| getIndex() | int | O(1) | Returns the zero-based index of the checkpoint in the parkour sequence. |
| clone() | Component | O(1) | Creates a shallow copy of the component. Used internally by the engine for entity duplication. |

## Integration Patterns

### Standard Usage
The component should never be used in isolation. It is designed to be retrieved from an entity that has been identified through another mechanism, such as a collision event.

```java
// A system processing a player's interaction with an entity
void onPlayerInteract(Player player, Entity entity) {
    // Retrieve the component from the entity, if it exists
    ParkourCheckpoint checkpoint = entity.getComponent(ParkourCheckpoint.getComponentType());

    if (checkpoint != null) {
        // Now, use the component's data to drive game logic
        int checkpointIndex = checkpoint.getIndex();
        playerParkourState.setCurrentCheckpoint(checkpointIndex);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Adding Logic:** Do not add methods containing game logic (e.g., `onPlayerReached()`) to this class. This violates the ECS principle of separating data from behavior and makes the system harder to maintain. All logic should reside in a dedicated *System*.
-   **Storing Volatile State:** Do not add fields to this component that represent temporary or runtime-only state. As this component is serialized, any state stored here will be persisted to disk, which can lead to bloated save files and complex state reconciliation issues on load.

## Data Pipeline
The ParkourCheckpoint primarily exists as data that flows through the engine's persistence and entity management systems.

> **Serialization Flow (World Save):**
> Entity in EntityStore -> **ParkourCheckpoint** instance -> `CODEC.encode()` -> Serialized Binary/Text Data -> World File on Disk

> **Deserialization Flow (World Load):**
> World File on Disk -> Serialized Binary/Text Data -> `CODEC.decode()` -> **ParkourCheckpoint** instance -> Attached to new Entity in EntityStore

> **Gameplay Logic Flow:**
> Collision Event -> Entity -> `EntityStore.getComponent()` -> **ParkourCheckpoint** instance -> ParkourSystem Logic

