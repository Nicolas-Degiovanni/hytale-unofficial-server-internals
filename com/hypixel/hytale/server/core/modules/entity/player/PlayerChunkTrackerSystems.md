---
description: Architectural reference for PlayerChunkTrackerSystems
---

# PlayerChunkTrackerSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** Utility

## Definition
```java
// Signature
public class PlayerChunkTrackerSystems {
    // Contains static inner System classes
}
```

## Architecture & Concepts
The PlayerChunkTrackerSystems class is not a traditional object but a static namespace for a collection of related systems within the server-side Entity Component System (ECS). These systems collectively manage the server's understanding of which world chunks each player entity is subscribed to. This is a fundamental mechanism for network culling and interest management, ensuring that clients only receive data for the parts of the world relevant to their player character.

This class embodies the ECS principle of separating data from behavior. The data is stored in the ChunkTracker component, while the logic that operates on that data is encapsulated within the two nested systems:

*   **AddSystem:** A one-shot system that initializes the ChunkTracker component the moment it is added to a player entity.
*   **UpdateSystem:** A per-tick system that continuously processes the ChunkTracker component to manage chunk loading, unloading, and visibility updates for the associated player.

These systems are registered with the world's system scheduler and are invoked automatically by the game engine. Developers do not interact with these classes directly; rather, they trigger their behavior by adding or removing a ChunkTracker component from an entity.

---

## AddSystem

**Type:** Transient System Logic

### Definition
```java
// Signature
public static class AddSystem extends HolderSystem<EntityStore> {
```

### Architecture & Concepts
The AddSystem is a reactive system that executes logic when an entity matching its query is added to the world. Its sole responsibility is to perform initial setup on a player's ChunkTracker component. By implementing HolderSystem, it hooks directly into the entity lifecycle, specifically the component addition event.

Its query targets any entity that has been given a ChunkTracker component. Upon detection, it immediately marks the component as ready, signaling to other parts of the engine that the chunk loading process can begin for this player.

### Lifecycle & Ownership
*   **Creation:** Instantiated once by the server's System Registry during world initialization. It is not created per-player.
*   **Scope:** Lives for the entire duration of the server world. It is a stateless, long-lived service object.
*   **Destruction:** De-registered and garbage collected when the world is shut down.

### Internal State & Concurrency
*   **State:** This system is entirely stateless. All state it manipulates is contained within the ChunkTracker component of the entity being processed.
*   **Thread Safety:** Operations are executed on the main server thread as part of the component modification command buffer processing. It is not designed for concurrent execution.

### API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns a query that selects for entities possessing a ChunkTracker component. |
| onEntityAdd(holder, reason, store) | void | O(1) | Callback invoked by the ECS when a matching entity is added. Sets the ChunkTracker to a "ready" state. |

---

## UpdateSystem

**Type:** Transient System Logic

### Definition
```java
// Signature
public static class UpdateSystem extends EntityTickingSystem<EntityStore> {
```

### Architecture & Concepts
The UpdateSystem is the primary workhorse for continuous chunk management. As an EntityTickingSystem, it is invoked by the engine on every server tick for every entity that possesses a ChunkTracker component.

Its core function is to delegate the per-tick logic to the ChunkTracker component itself via the component's tick method. This is where complex calculations for determining a player's view distance, chunk subscription changes, and network packet generation are orchestrated. The system acts as the bridge between the engine's main loop and the component's specific logic.

### Lifecycle & Ownership
*   **Creation:** Instantiated once by the server's System Registry during world initialization.
*   **Scope:** Persists for the lifetime of the server world.
*   **Destruction:** Cleaned up when the world is shut down.

### Internal State & Concurrency
*   **State:** This system is stateless. All relevant state, such as the player's current position and subscribed chunks, is stored within the ChunkTracker component on each entity.
*   **Thread Safety:** This system is **explicitly not thread-safe**. The isParallel method returns false, which is a critical directive to the engine's scheduler. This forces the engine to process all entities with a ChunkTracker component serially, on a single thread. This design implies that the underlying logic within ChunkTracker.tick modifies shared state or is otherwise not safe for concurrent execution.

**WARNING:** Modifying isParallel to return true without a complete rewrite of the underlying ChunkTracker logic will introduce severe race conditions and data corruption.

### API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns a query that selects for entities possessing a ChunkTracker component. |
| isParallel(...) | boolean | O(1) | Returns false, indicating that this system's tick logic must be executed serially. |
| tick(dt, index, chunk, store, buffer) | void | O(N) | Callback invoked by the ECS each tick. Delegates the update logic to the ChunkTracker component. |

## Integration Patterns

### Standard Usage
A developer never calls these systems directly. The entire interaction is managed by the ECS framework. To make a player entity manage chunks, simply add the ChunkTracker component to it.

```java
// Correctly engage the PlayerChunkTrackerSystems
// Assume 'playerEntityRef' is a valid reference to a player entity
// and 'commandBuffer' is an active command buffer.

// This action will trigger AddSystem.onEntityAdd
commandBuffer.addComponent(playerEntityRef, new ChunkTracker());

// On subsequent server ticks, UpdateSystem.tick will be called for this entity.
```

### Anti-Patterns (Do NOT do this)
*   **Direct Instantiation:** Do not use `new PlayerChunkTrackerSystems.UpdateSystem()`. The systems are managed by the engine's System Registry. Manually creating an instance will have no effect and will not be registered for updates.
*   **Manual Invocation:** Do not call the `tick` or `onEntityAdd` methods directly. This bypasses the engine's scheduler and state management, leading to unpredictable behavior, state corruption, and potential crashes.
*   **Stateful Systems:** Do not add member variables or any other state to these system classes. They are designed to be stateless processors; all state must reside in components.

## Data Pipeline
The flow of control and data is dictated by the server's main loop and the ECS framework.

> **Initialization Flow:**
> Entity Creation -> CommandBuffer.addComponent(ChunkTracker) -> ECS Framework -> **AddSystem.onEntityAdd** -> ChunkTracker state is initialized

> **Update Flow:**
> Server Tick -> System Scheduler -> **UpdateSystem.tick** (for each player) -> ChunkTracker.tick -> Generates network commands -> Network Layer -> Client receives chunk updates

