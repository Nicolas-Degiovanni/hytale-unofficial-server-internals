---
description: Architectural reference for DeleteCursedItemsOnSpawnSystem
---

# DeleteCursedItemsOnSpawnSystem

**Package:** com.hypixel.hytale.builtin.portals.systems.curse
**Type:** Transient

## Definition
```java
// Signature
public class DeleteCursedItemsOnSpawnSystem extends RefSystem<EntityStore> {
```

## Architecture & Concepts
The DeleteCursedItemsOnSpawnSystem is a reactive, rule-enforcing component within the Entity Component System (ECS) framework. Its primary architectural function is to maintain game state integrity by cleansing a player's inventory under specific world-transition conditions.

This system subscribes to entity creation events, but its query is filtered to only trigger when an entity with a **Player** component is added to the world. This effectively hooks into the player spawn lifecycle.

Upon activation, the system performs a critical check: it queries for the existence of a global **PortalWorld** resource. If this resource is *not* present, the system infers that the player is spawning into a standard, non-portal dimension. In this scenario, it invokes the **CursedItems** utility to forcibly remove all "cursed" items from the player's inventory. This prevents players from carrying potentially game-breaking or dimension-specific items into worlds where they are not intended to exist.

## Lifecycle & Ownership
- **Creation:** This system is not instantiated directly by game logic. It is discovered and instantiated by the server's central **SystemManager** during the world initialization phase. The engine is the sole creator and owner of system instances.
- **Scope:** An instance of this system persists for the entire lifetime of the game world it is registered with. It is a long-lived, stateless service.
- **Destruction:** The instance is marked for garbage collection when the associated game world is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no member fields and all operations are performed on data passed into its methods by the ECS scheduler (specifically the Store and Ref objects). Its behavior is entirely deterministic based on its inputs.
- **Thread Safety:** The system instance itself is inherently thread-safe due to its stateless nature. However, its methods are designed to be called exclusively by the engine's main game loop thread for a given world. All interactions with the **Store** and **CommandBuffer** are guaranteed by the engine to be synchronized. Manual invocation from other threads will lead to race conditions and world state corruption.

## API Surface
The public contract is defined by its role as a RefSystem. Developers do not call these methods directly; the engine's scheduler invokes them.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdded(...) | void | O(N) | Engine callback. Triggers when a Player entity spawns. Complexity is O(N) where N is the number of items in the player's inventory. |
| onEntityRemove(...) | void | O(1) | Engine callback. A no-op in this implementation. |
| getQuery() | Query | O(1) | Configures the system to only listen for entities with a Player component. |

## Integration Patterns

### Standard Usage
This system is not used directly in code. It is automatically registered with the engine's SystemManager at startup. Its logic is applied transparently to all players. No developer interaction is required for it to function.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new DeleteCursedItemsOnSpawnSystem()`. The object will not be registered with the engine and its callback methods will never be invoked.
- **Manual Invocation:** Do not call `onEntityAdded` from your own code. This bypasses the transactional and state management guarantees of the ECS engine and can corrupt world state.

## Data Pipeline
This system acts as a conditional filter in the player spawn data pipeline.

> Flow:
> Player joins server -> Engine creates Player entity -> ECS Scheduler dispatches `onEntityAdded` event -> **DeleteCursedItemsOnSpawnSystem** receives event -> System reads `PortalWorld` resource state -> If resource is absent, system modifies Player inventory state.

