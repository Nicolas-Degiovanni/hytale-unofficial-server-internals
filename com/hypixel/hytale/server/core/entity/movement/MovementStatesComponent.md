---
description: Architectural reference for MovementStatesComponent
---

# MovementStatesComponent

**Package:** com.hypixel.hytale.server.core.entity.movement
**Type:** Data Component

## Definition
```java
// Signature
public class MovementStatesComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The MovementStatesComponent is a fundamental data container within the server-side Entity-Component-System (ECS) architecture. Its sole responsibility is to hold the server-authoritative movement state for an associated entity. This component strictly adheres to the ECS principle of separating data (Components) from logic (Systems), acting as a plain data object manipulated by various server systems.

The core architectural feature is its dual-state tracking mechanism, embodied by the `movementStates` and `sentMovementStates` fields.

1.  **movementStates**: Represents the *current, real-time* state of the entity as determined by the server's game logic, physics, or AI systems. This is the source of truth for all server-side calculations.
2.  **sentMovementStates**: Acts as a snapshot of the last state that was successfully synchronized and sent to the client.

This dual-state design is critical for network optimization. A dedicated network synchronization system compares the current `movementStates` against the last `sentMovementStates` on each tick. If and only if a change is detected, a network packet is dispatched. This prevents redundant data transmission, conserving server bandwidth.

## Lifecycle & Ownership
-   **Creation:** A MovementStatesComponent is instantiated and attached to an Entity by a higher-level authority, such as an `EntityManager` or `EntityFactory`. This typically occurs when an entity that can move (like a player or a mob) is spawned into the world. The component's type is resolved via the central `EntityModule`.
-   **Scope:** The component's lifecycle is strictly bound to its parent Entity. It persists as long as the Entity exists within the `EntityStore`.
-   **Destruction:** The component is marked for garbage collection when its parent Entity is removed from the world. There is no manual destruction method; its lifetime is managed entirely by the ECS framework.

## Internal State & Concurrency
-   **State:** This component is highly **mutable**. Its internal `MovementStates` objects are designed to be frequently updated by server-side systems during the game loop. It serves as a volatile scratchpad for an entity's current actions.
-   **Thread Safety:** **This component is not thread-safe.** It is designed for single-threaded access within the main server game tick. Any modification or access from asynchronous tasks, network threads, or worker pools without explicit external locking will result in data corruption, race conditions, and unpredictable entity behavior. All interactions must be scheduled to run on the main game thread.

## API Surface
The public contract is minimal, consisting primarily of accessors and a mechanism for cloning.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component from the central `EntityModule`. |
| getMovementStates() | MovementStates | O(1) | Returns the current, server-authoritative movement state. |
| setMovementStates(states) | void | O(1) | Overwrites the current movement state. Typically called by physics or AI systems. |
| getSentMovementStates() | MovementStates | O(1) | Returns the last movement state that was synchronized with the client. |
| setSentMovementStates(states) | void | O(1) | Overwrites the last-sent state. **Warning:** This should only be called by the network layer. |
| clone() | Component | O(N) | Creates a deep copy of the component and its internal state. |

## Integration Patterns

### Standard Usage
A server-side *System* (e.g., `PlayerMovementSystem`) queries for entities with this component, reads player input, and updates the `movementStates` field. A separate `NetworkSyncSystem` later reads this value.

```java
// Example from within a server-side System's update method
Entity playerEntity = world.getEntity(playerId);
MovementStatesComponent moveComp = playerEntity.getComponent(MovementStatesComponent.getComponentType());

// Logic to process input and update the entity's state
MovementStates newStates = moveComp.getMovementStates();
newStates.setWalking(true);
newStates.setSprinting(false);

// The component is now dirty and will be processed by the network system
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new MovementStatesComponent()`. Components must be added to an entity via the `EntityManager` or `Entity.addComponent()` API to ensure they are correctly registered and tracked by the game engine.
-   **Logic System State Modification:** Game logic systems (AI, Physics) must never modify the `sentMovementStates` field. This field is owned exclusively by the network synchronization layer. Modifying it directly will break the state diffing mechanism, causing network desynchronization.
-   **Asynchronous Access:** Do not read or write to this component from another thread. Schedule a task to run on the main game thread to ensure data consistency.

## Data Pipeline
The component acts as a stateful bridge between game logic and the network layer.

> Flow:
> AI / Physics System -> **MovementStatesComponent.movementStates** (Write) -> Network Sync System (Read & Compare) -> **MovementStatesComponent.sentMovementStates** (Write) -> Network Packet Encoder -> Client

