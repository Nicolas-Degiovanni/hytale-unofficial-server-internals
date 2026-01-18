---
description: Architectural reference for CombatViewSystems
---

# CombatViewSystems

**Package:** com.hypixel.hytale.server.npc.blackboard.view.combat
**Type:** Utility

## Definition
```java
// Signature
public class CombatViewSystems {
    // This class is a non-instantiable container for related ECS types.
    // It exposes static methods and nests the following key types:
    // - public static class CombatData implements Component
    // - public static class CombatDataPool implements Resource
    // - public static class Ensure extends HolderSystem
    // - public static class EntityRemoved extends HolderSystem
    // - public static class Ticking extends EntityTickingSystem
}
```

## Architecture & Concepts
CombatViewSystems is not a single object but a cohesive subsystem designed to provide a high-level, interpreted view of an entity's combat state. It acts as a translation layer, converting raw, low-level interaction data from the InteractionManager into a simplified, queryable format suitable for high-level logic, particularly NPC AI and behavior trees.

The subsystem is built on a lazy-evaluation and per-frame caching model to optimize performance. The expensive process of analyzing an entity's active interactions is deferred until the data is first requested within a game tick. Subsequent requests within the same tick receive the cached result.

This subsystem is composed of five key types working in concert:
*   **CombatData (Component):** A per-entity data structure attached to game objects. It stores the cached list of interpreted combat actions and a dirty flag to manage the cache state.
*   **CombatDataPool (Resource):** A world-scoped object pool for InterpretedCombatData instances. This is a critical performance optimization that significantly reduces garbage collection pressure by recycling temporary data objects each frame.
*   **Ensure (System):** An ECS system that automatically attaches the CombatData component to all newly created entities, guaranteeing its presence for any entity that might engage in combat.
*   **Ticking (System):** An ECS system that runs once per game tick. Its sole responsibility is to invalidate the cache for all entities by clearing the previously interpreted data. This ensures data is never stale across frames.
*   **EntityRemoved (System):** An ECS system that cleans up memory when an entity is destroyed, ensuring all its associated InterpretedCombatData objects are returned to the CombatDataPool.

### Lifecycle & Ownership
The lifecycle of this subsystem is intrinsically tied to the server's core Entity-Component-System (ECS) world.

*   **Creation:** The three systems (Ensure, Ticking, EntityRemoved) and the CombatDataPool resource are instantiated and registered with the ECS world during server initialization, likely by the NPCPlugin. The CombatData component is not created at this time; it is added to entities dynamically by the Ensure system as they spawn.
*   **Scope:** The systems and the CombatDataPool resource are singletons that persist for the entire lifetime of the server's ECS world. The CombatData component's lifetime is bound to its parent entity.
*   **Destruction:** The CombatData component is destroyed when its parent entity is removed from the world. The EntityRemoved system intercepts this event to perform critical cleanup, preventing memory leaks by returning objects to the pool. The systems and the resource are de-registered and garbage collected when the server world is shut down.

## Internal State & Concurrency
*   **State:** State is primarily managed in two places: the per-entity CombatData component and the global CombatDataPool resource. The CombatData component is highly mutable, with its contents being cleared and re-populated every game tick. The `interpreted` boolean flag within it acts as a cache-validity marker. The CombatDataPool is also mutable, as its internal collection of pooled objects changes constantly.
*   **Thread Safety:** This subsystem is **not thread-safe** and is designed for single-threaded access from the main server game loop. The Ticking system explicitly declares `isParallel(false)`, enforcing that its cache-invalidation logic runs serially. Calling getCombatData from an asynchronous task or a different thread will lead to unpredictable behavior, including stale data reads and `ConcurrentModificationException`.

## API Surface
The primary public contract is the static `getCombatData` method. The nested classes are public for registration with the ECS framework but are not intended for direct manipulation by client code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCombatData(Ref) | List<InterpretedCombatData> | O(N) first call, O(1) subsequent | Lazily computes and returns a read-only list of the entity's current combat actions. N is the number of active interactions. Throws if the entity does not exist. |

## Integration Patterns

### Standard Usage
This system is designed to be queried by other server-side systems, such as AI behavior trees, to make decisions based on an entity's current actions.

```java
// Example: Inside an AI Behavior Tree node or other server logic
Ref<EntityStore> npcRef = ...; // Reference to the NPC entity

// This call is cheap after the first request in a frame.
List<InterpretedCombatData> combatActions = CombatViewSystems.getCombatData(npcRef);

if (!combatActions.isEmpty()) {
    InterpretedCombatData primaryAction = combatActions.get(0);
    if (primaryAction.isPerformingMeleeAttack()) {
        // The NPC is currently attacking, perhaps we should not interrupt it.
    } else if (primaryAction.isPerformingBlock()) {
        // The NPC is blocking, find a way to break its guard.
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new CombatData()` or `new Ticking()`. These objects are exclusively managed by the ECS framework. Manually creating them will have no effect and will not be integrated into the engine's lifecycle.
-   **Manual Cache Management:** Do not attempt to manually clear the list within a CombatData component or modify its `interpreted` flag. The Ticking system's cache invalidation is fundamental to the subsystem's correctness. Interfering will cause stale data and unpredictable AI behavior.
-   **Multi-threaded Access:** **CRITICAL:** Do not call getCombatData from a worker thread or any context outside the main server tick. The underlying data structures are not synchronized, and doing so will result in data corruption and server instability.

## Data Pipeline
The flow of data through this system is cyclical, resetting each game tick. It transforms raw engine state into actionable intelligence for AI.

> Flow:
> Raw InteractionManager State -> `getCombatData` Request -> **CombatViewSystems Interpretation** (uses CombatDataPool) -> Cached `InterpretedCombatData` List -> AI System (reads list) -> AI Decision

