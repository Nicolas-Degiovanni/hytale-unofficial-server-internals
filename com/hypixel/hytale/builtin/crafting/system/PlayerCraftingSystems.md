---
description: Architectural reference for PlayerCraftingSystems
---

# PlayerCraftingSystems

**Package:** com.hypixel.hytale.builtin.crafting.system
**Type:** Utility

## Definition
```java
// Signature
public class PlayerCraftingSystems {
    // Contains nested static system classes
}
```

## Architecture & Concepts
The PlayerCraftingSystems class is not a system itself, but a static container or namespace for a suite of related Entity Component Systems (ECS). This pattern promotes high cohesion by grouping systems that operate on the same domain—in this case, the crafting functionality associated with a Player entity.

It encapsulates two distinct but cooperative systems:
1.  **CraftingManagerAddSystem:** A reactive system responsible for the lifecycle of the CraftingManager component. It ensures the component is present on all player entities and handles its cleanup.
2.  **PlayerCraftingSystem:** A processing system that executes logic every game tick for entities that possess a CraftingManager component.

This separation follows standard ECS design principles, isolating lifecycle management from per-tick processing logic.

---
description: Architectural reference for PlayerCraftingSystems.CraftingManagerAddSystem
---

# PlayerCraftingSystems.CraftingManagerAddSystem

**Package:** com.hypixel.hytale.builtin.crafting.system
**Type:** Transient

## Definition
```java
// Signature
public static class CraftingManagerAddSystem extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
CraftingManagerAddSystem is a **reactive lifecycle system**. Its primary function is to manage the existence of the CraftingManager component on entities. It does not run on every game tick. Instead, the ECS framework invokes its methods in response to specific entity lifecycle events.

The system guarantees that any entity identified as a Player will automatically be equipped with a CraftingManager component. This enforces a critical architectural invariant: a Player *always* has the capability to craft.

Furthermore, it acts as a cleanup mechanism. Upon player entity removal, it intercepts the event to safely terminate any ongoing crafting processes and trigger a player data save, preventing data loss and orphaned processes.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's core SystemGraph during world initialization. A single instance is created and registered to listen for entity events.
-   **Scope:** Session-scoped. The system persists for the entire lifetime of the game world.
-   **Destruction:** The instance is discarded when the world is unloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** The system is effectively stateless. It holds immutable references to ComponentType definitions, which act as handles for querying the ECS data store. It does not cache entity data.
-   **Thread Safety:** This system is not thread-safe. The ECS framework guarantees that its `onEntityAdd` and `onEntityRemoved` methods are invoked from a single, controlled thread, preventing race conditions related to entity modification. Direct concurrent invocation is unsupported and will lead to undefined behavior.

## API Surface
The public API consists of callbacks defined by the HolderSystem base class. These are intended for engine invocation only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(1) | Engine callback. Ensures a CraftingManager component exists on the entity. |
| onEntityRemoved(holder, reason, store) | void | O(N) | Engine callback. Cancels all active crafts (N = number of crafts) and saves player data. |
| getQuery() | Query | O(1) | Returns a query that selects all entities with a Player component. |

## Integration Patterns

### Standard Usage
This system is not used directly by feature developers. It is automatically registered with the engine's SystemGraph at startup. Its operation is transparent.

### Anti-Patterns (Do NOT do this)
-   **Manual Invocation:** Never call `onEntityAdd` or `onEntityRemoved` directly. Doing so bypasses the engine's state management and event queue, which can corrupt entity state.
-   **Direct Instantiation:** Do not create instances of this system using `new`. System creation and registration is exclusively managed by the engine's dependency injection and world setup process.

## Data Pipeline
This system responds to events in the entity data pipeline.

> **Creation Flow:**
> New Entity with Player component added -> ECS Framework Event -> **CraftingManagerAddSystem.onEntityAdd** -> Command to add CraftingManager component

> **Destruction Flow:**
> Entity with Player component removed -> ECS Framework Event -> **CraftingManagerAddSystem.onEntityRemoved** -> Cancels active crafts -> Triggers player save operation

---
description: Architectural reference for PlayerCraftingSystems.PlayerCraftingSystem
---

# PlayerCraftingSystems.PlayerCraftingSystem

**Package:** com.hypixel.hytale.builtin.crafting.system
**Type:** Transient

## Definition
```java
// Signature
public static class PlayerCraftingSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
PlayerCraftingSystem is a **parallelizable processing system**. It is the workhorse that drives crafting progression over time. On every server tick, this system iterates over all entities that have a CraftingManager component and invokes their `tick` method.

This design cleanly separates the "what" (the data in the CraftingManager component) from the "how" (the logic in this system). The system acts as a simple, efficient dispatcher, delegating the complex state-machine logic of crafting to the component itself.

Crucially, the system declares its potential for parallel execution via the `isParallel` method. The engine's scheduler can leverage this to distribute the workload of ticking many players' crafting managers across multiple CPU cores, significantly improving server performance under load.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's core SystemGraph during world initialization.
-   **Scope:** Session-scoped. The system persists for the entire lifetime of the game world and is active in every game tick.
-   **Destruction:** The instance is discarded when the world is unloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** This system is entirely stateless. It holds no data between ticks besides an immutable reference to the CraftingManager ComponentType.
-   **Thread Safety:** The `tick` method is designed for concurrent execution. The engine's scheduler guarantees that each invocation of `tick` operates on a unique entity. Because the system itself is stateless and each component is self-contained, there is no risk of data races between threads processing different entities.

## API Surface
The public API consists of callbacks defined by the EntityTickingSystem base class. These are intended for engine invocation only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, buffer) | void | O(1) | Engine callback. Invoked once per entity per tick. Delegates logic to CraftingManager.tick. |
| getQuery() | Query | O(1) | Returns a query that selects all entities with a CraftingManager component. |
| isParallel(chunkSize, taskCount) | boolean | O(1) | Signals to the engine that this system's workload can be parallelized. |

## Integration Patterns

### Standard Usage
This system is automatically registered with the engine's SystemGraph. A developer interacts with it indirectly by adding or modifying a CraftingManager component on an entity.

```java
// A developer does not call this system.
// The engine's main loop executes it like this:

// 1. Scheduler finds all entities matching the query.
// 2. For each entity, it calls:
//    playerCraftingSystem.tick(...);
```

### Anti-Patterns (Do NOT do this)
-   **Manual Ticking:** Do not call the `tick` method from other systems or game logic. This subverts the main game loop and its performance schedulers, potentially causing double-ticking or threading violations.
-   **Stateful Implementation:** Do not add mutable fields to this class. Systems should remain stateless to ensure correctness, especially when running in parallel.

## Data Pipeline
This system is a processor in the main game loop's data pipeline.

> **Flow:**
> Game Tick Start -> ECS Scheduler -> **PlayerCraftingSystem.tick** (for each relevant entity) -> CraftingManager.tick() -> Component State Update (e.g., progress timer) -> Command Buffer (for state changes like item creation)

