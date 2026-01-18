---
description: Architectural reference for PlayerSendInventorySystem
---

# PlayerSendInventorySystem

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** Transient System

## Definition
```java
// Signature
public class PlayerSendInventorySystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The PlayerSendInventorySystem is a server-side system within the Entity Component System (ECS) framework. Its sole responsibility is to synchronize the state of a player's inventory with their connected client. It functions as a critical bridge between the in-memory game state and the network layer.

This system operates on a specific set of entities defined by its internal **Query**. It targets any entity that possesses both a **Player** component (containing the core player data, including the Inventory) and a **PlayerRef** component (containing the network connection handle).

During each server tick, this system iterates over all matching player entities. For each one, it checks a "dirty" flag on the player's Inventory object. If the inventory has been modified since the last check, the system serializes the full inventory state into a network packet and dispatches it to the corresponding client via the PacketHandler. This "dirty flag" mechanism is a common and efficient optimization pattern, ensuring that network traffic is only generated when state has actually changed.

## Lifecycle & Ownership
-   **Creation:** An instance of PlayerSendInventorySystem is created and registered with the server's central system scheduler during the world initialization phase. It is not instantiated on-demand or per-player.
-   **Scope:** The system has a global scope and persists for the entire lifetime of the server's game world. It is a long-lived object that continuously operates on the entire set of connected players.
-   **Destruction:** The system is destroyed and de-registered when the server world is shut down.

## Internal State & Concurrency
-   **State:** This class is fundamentally stateless. Its member fields are final references to component types and the pre-built query, established at construction. It does not store or cache any per-player data between ticks. All state is read directly from the components of the entities it processes.
-   **Thread Safety:** The system is designed for parallel execution. The `isParallel` method allows the ECS scheduler to run the `tick` method for different chunks of entities across multiple threads simultaneously. The core ECS architecture guarantees that a single thread will operate on a given `ArchetypeChunk`, preventing data races on the components being read. The use of `consumeIsDirty` is an atomic operation that ensures the dirty flag is correctly handled even in a multi-threaded environment.

## API Surface
The public API is minimal, as the system is designed to be controlled exclusively by the ECS scheduler, not by general application code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the pre-compiled query used by the scheduler to find matching entities. |
| isParallel(int, int) | boolean | O(1) | A hint to the scheduler, indicating if this system's workload can be parallelized. |
| tick(float, int, ArchetypeChunk, Store, CommandBuffer) | void | O(1) per entity | The core logic executed by the scheduler for each entity matching the query. |

## Integration Patterns

### Standard Usage
A developer does not interact with this system directly. It is registered once with the main world or system manager during server startup. The engine's game loop is then responsible for invoking its `tick` method on every frame for all relevant entities.

```java
// Example of system registration during server initialization
SystemManager manager = server.getSystemManager();
ComponentType<EntityStore, Player> playerType = ...;

// The system is instantiated and added to the scheduler
manager.addSystem(new PlayerSendInventorySystem(playerType));
```

### Anti-Patterns (Do NOT do this)
-   **Manual Invocation:** Never call the `tick` method directly. Doing so bypasses the ECS scheduler, its query optimizations, and its parallel execution capabilities, which can lead to severe performance degradation and race conditions.
-   **Stateful Implementation:** Do not add mutable member variables to this class to store temporary data. Systems must remain stateless to ensure they are thread-safe and behave predictably.
-   **Direct Instantiation:** Do not create instances of this class using `new` in general game logic. It should only be instantiated once during the server's bootstrap sequence.

## Data Pipeline
This system acts as a synchronization point in the data flow, triggered by changes to a player's inventory.

> Flow:
> Other Game System (e.g., item pickup) -> Modifies `Inventory` component -> `Inventory.isDirty` flag set to true -> ECS Scheduler executes **PlayerSendInventorySystem** -> `consumeIsDirty` returns true -> `Inventory` is serialized to a packet -> `PacketHandler.write` is called -> Network Packet sent to Client

