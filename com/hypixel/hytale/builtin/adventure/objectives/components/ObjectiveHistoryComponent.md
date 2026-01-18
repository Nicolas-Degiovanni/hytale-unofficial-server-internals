---
description: Architectural reference for ObjectiveHistoryComponent
---

# ObjectiveHistoryComponent

**Package:** com.hypixel.hytale.builtin.adventure.objectives.components
**Type:** Data Component

## Definition
```java
// Signature
public class ObjectiveHistoryComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The ObjectiveHistoryComponent is a data-only component within the server-side Entity-Component-System (ECS) architecture. Its sole responsibility is to attach persistent state to an Entity, typically a player, to track their progress through adventure mode objectives.

It functions as a simple data container, holding two maps that associate objective identifiers (as Strings) with their corresponding historical data. This component does not contain any logic; it is manipulated exclusively by other systems, such as an ObjectiveSystem, which are responsible for game logic.

A critical architectural feature is the static **CODEC** field. This serialization contract allows the Hytale engine to transparently save and load the component's state as part of the world's `EntityStore`. This mechanism ensures that a player's objective progress persists across game sessions without requiring custom save/load logic in higher-level game systems.

## Lifecycle & Ownership
- **Creation:** An ObjectiveHistoryComponent is never instantiated directly. It is created and attached to an Entity under two conditions:
    1.  By a game system when an Entity first interacts with an objective that requires historical tracking.
    2.  By the `EntityStore` during world loading, when deserializing an Entity that previously had this component.
- **Scope:** The component's lifetime is strictly bound to the lifetime of the Entity it is attached to. It persists as long as the owning Entity exists in the world.
- **Destruction:** The component is marked for garbage collection when its parent Entity is destroyed or when a system explicitly removes the component from the Entity.

## Internal State & Concurrency
- **State:** The internal state is **highly mutable**. The component's primary purpose is to have its internal maps modified by game systems to reflect real-time changes in objective completion. It effectively acts as a server-side cache for a player's quest progress.

- **Thread Safety:** This component is **not thread-safe** and must only be accessed from the main server thread that processes the owning Entity. The use of `Object2ObjectOpenHashMap` from the fastutil library is a deliberate performance choice that offers no concurrency guarantees. Unsynchronized access from multiple threads will lead to data corruption and server instability.

## API Surface
The public API is minimal, providing direct access to the underlying data maps and a cloning mechanism for the ECS framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getObjectiveHistoryMap() | Map | O(1) | Returns a reference to the internal map of objective history. |
| getObjectiveLineHistoryMap() | Map | O(1) | Returns a reference to the internal map of objective line history. |
| clone() | Component | O(N+M) | Creates a deep copy of the component and its internal maps. Used by the engine for entity duplication or state snapshotting. |

## Integration Patterns

### Standard Usage
Systems should retrieve this component from an entity to read or write objective progress within a single game tick.

```java
// A system retrieves the component from an entity to update its state
Entity playerEntity = ...;
ObjectiveHistoryComponent history = playerEntity.getComponent(ObjectiveHistoryComponent.class);

if (history != null) {
    Map<String, ObjectiveHistoryData> progressMap = history.getObjectiveHistoryMap();
    progressMap.put("main_quest_01", new ObjectiveHistoryData(...));
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ObjectiveHistoryComponent()`. The ECS framework and `EntityStore` are solely responsible for its lifecycle. Direct creation bypasses the engine's management and persistence systems.
- **Cross-Thread Modification:** Do not access or modify the component from any thread other than the primary server game loop. This will cause severe race conditions.
- **Retaining References:** Avoid storing a reference to this component or its internal maps across multiple ticks. Always re-fetch the component from the entity each time it is needed to ensure you are working with the most current state.

## Data Pipeline
The ObjectiveHistoryComponent is a state container that sits between game logic and the persistence layer.

> **Runtime Flow:**
> Game Event (e.g., Quest Step Completed) -> ObjectiveSystem -> **ObjectiveHistoryComponent** (State is mutated)

> **Persistence Flow (Saving):**
> **ObjectiveHistoryComponent** (on Entity) -> EntityStore Snapshot -> CODEC Serialization -> World Save (Disk)

> **Persistence Flow (Loading):**
> World Save (Disk) -> CODEC Deserialization -> **ObjectiveHistoryComponent** (Instantiated on Entity) -> Game Systems (Ready for access)

