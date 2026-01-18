---
description: Architectural reference for PositionCacheSystems
---

# PositionCacheSystems

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** Utility / Namespace

## Definition
```java
// Signature
public class PositionCacheSystems {
    // Contains only static methods and nested static System classes.
    // Not designed for instantiation.
}
```

## Architecture & Concepts

PositionCacheSystems is a critical component of the server-side NPC artificial intelligence framework. It is not a single, instantiable system but rather a namespace for a collection of related Entity-Component-Systems (ECS) that collectively manage the **PositionCache** for every NPC.

The primary architectural purpose of this class and its nested systems is to act as the sensory input layer for NPC decision-making. High-frequency spatial queries (e.g., "find all players within 10 meters") are computationally expensive. Instead of allowing every individual AI behavior to perform these queries on demand, PositionCacheSystems pre-fetches and caches this spatial data once per update cycle. AI behaviors then read from this cheap, local cache, dramatically improving server performance.

The systems within this class are responsible for two key functions:

1.  **Initialization:** The `RoleActivateSystem` and `OnFlockJoinSystem` react to NPC creation and state changes. They use the static `initialisePositionCache` method to configure an NPC's `PositionCache` based on the requirements of its assigned `Role`, `Instruction`s, and `StateEvaluator`. This configuration defines *what* data the NPC needs (e.g., players within 20m, other NPCs within 5m).

2.  **Population:** The `UpdateSystem` runs on a regular tick. It iterates through all active NPCs, executes the spatial queries defined during initialization, and populates the `PositionCache` with the results. This ensures the NPC's sensory data is kept reasonably up-to-date.

In essence, PositionCacheSystems decouples AI logic from the expensive process of world-state querying, acting as a centralized and optimized data provider for NPC perception.

### Lifecycle & Ownership
-   **Creation:** The nested system classes (`UpdateSystem`, `RoleActivateSystem`, `OnFlockJoinSystem`) are instantiated by the server's central ECS scheduler during plugin initialization, typically as part of the `NPCPlugin` bootstrap sequence. The top-level `PositionCacheSystems` class is never instantiated.
-   **Scope:** The system instances persist for the entire lifetime of the server. They are stateless services that operate on component data. The `PositionCache` component itself, which holds the data, has its lifecycle tied to its parent NPC entity.
-   **Destruction:** The systems are discarded upon server shutdown. When an NPC entity is removed from the world, the `RoleActivateSystem`'s `onEntityRemoved` callback ensures its associated `PositionCache` is reset to release memory and references.

## Internal State & Concurrency
-   **State:** The `PositionCacheSystems` class and its nested systems are fundamentally stateless. All state they manipulate is external, residing within the `PositionCache` component of individual NPC entities or within global `SpatialResource`s managed by the ECS framework.

-   **Thread Safety:** This system collection is **not thread-safe** for parallel execution and is designed with specific concurrency constraints.
    -   The `UpdateSystem` explicitly returns `false` from its `isParallel` method. This is a critical design choice, forcing the ECS scheduler to execute all updates for this system on a single thread. This prevents data races when querying the shared, global `SpatialResource` collections for players, NPCs, and items.
    -   The use of `ThreadLocal` for `BUCKET_POOL_THREAD_LOCAL` is a standard engine pattern to provide thread-specific temporary collections for query results, avoiding both heap allocation overhead and cross-thread contamination if the engine were to use a worker thread pool for other systems.

    **WARNING:** Any attempt to modify the `isParallel` flag or manually schedule the `UpdateSystem` across multiple threads will lead to severe data corruption and server instability.

## API Surface

The primary public contract is the static initialization method. The nested systems are considered internal machinery of the ECS framework and should not be invoked directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| initialisePositionCache(Role, StateEvaluator, double) | void | O(N) | Configures a `PositionCache` based on an NPC's `Role`. Complexity is O(N) where N is the number of behaviors and state transitions registered in the `Role` that require spatial data. This method is the central point of configuration for an NPC's sensory needs. |

## Integration Patterns

### Standard Usage

Developers do not interact with these systems directly. Instead, they configure an NPC's `Role` component. The systems automatically interpret this configuration at runtime.

```java
// In an NPC's archetype definition or setup logic:
NPCEntity npc = ...;
Role role = npc.getRole();

// 1. Configure the Role with behaviors that require spatial awareness.
role.setAvoidingEntities(true);
role.setSeparationDistance(5.0);

// 2. When the NPC is added to the world, RoleActivateSystem automatically runs.
// 3. It calls initialisePositionCache, which reads the Role's configuration.
// 4. The PositionCache is now configured to query for entities within 5.0 meters.
// 5. On each tick, UpdateSystem will populate the cache with this data.
```

### Anti-Patterns (Do NOT do this)
-   **Direct System Invocation:** Never get an instance of `UpdateSystem` from the engine and call its `steppedTick` method. This bypasses the dependency scheduler (`getDependencies`), leading to race conditions where the `PositionCache` is updated before or after the spatial resources themselves are updated, resulting in off-by-one-tick errors in AI perception.
-   **Post-Initialization Role Mutation:** Modifying an NPC's `Role` properties (e.g., changing `separationDistance`) after the NPC has been spawned will not automatically re-configure the `PositionCache`. The cache will continue to operate on its initial configuration, leading to behavior that does not match the new `Role` settings. A full re-initialization must be triggered.

## Data Pipeline

The `UpdateSystem` is responsible for a one-way data flow from global spatial indexes into the local `PositionCache` of each NPC. This pipeline refreshes the NPC's perception of the world.

> Flow:
> Global `SpatialResource` (Player Positions) -> **`PositionCacheSystems.UpdateSystem`** (Queries for nearby entities) -> `PositionCache` Component (Writes results) -> AI Behavior Systems (Reads from cache)

