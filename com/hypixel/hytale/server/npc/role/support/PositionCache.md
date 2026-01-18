---
description: Architectural reference for PositionCache
---

# PositionCache

**Package:** com.hypixel.hytale.server.npc.role.support
**Type:** Transient

## Definition
```java
// Signature
public class PositionCache {
```

## Architecture & Concepts

The PositionCache is a critical performance and organizational component within the server-side NPC AI framework. It functions as the primary sensory input and perception layer for an individual NPC's behavior, as defined by its parent **Role**.

Architecturally, this class decouples AI decision-making logic from the expensive process of world-state querying. Instead of each AI behavior independently scanning the entire world for entities every tick, the PositionCache performs a single, optimized query and caches the results. This cache is then consumed by various AI sensors and behaviors associated with the parent Role.

The system is designed around a "configuration-then-execution" model. During an initial setup phase, various AI behaviors register their maximum perception distances using methods like `requirePlayerDistanceSorted`. This allows the PositionCache to build a master query that is broad enough to satisfy all needs without over-fetching data. Once configured, it enters an operational state, periodically refreshing its internal lists of nearby entities based on a time-to-live (TTL) mechanism.

Key responsibilities include:
-   Maintaining bucketed and sorted lists of nearby players and NPCs.
-   Caching the results of expensive Line of Sight (LOS) calculations.
-   Providing efficient query methods to find the closest entities that match specific criteria.
-   Tracking other relevant world objects like dropped items and spawn markers.

## Lifecycle & Ownership

-   **Creation:** A PositionCache instance is created and owned exclusively by a `Role` object. It is instantiated directly within the `Role`'s constructor, establishing a tight, 1:1 coupling.
-   **Scope:** The lifecycle of a PositionCache is identical to that of its parent `Role`. It persists as long as the `Role` is active on an NPC.
-   **Destruction:** The object is eligible for garbage collection when its parent `Role` is destroyed. The `reset` method allows the cache to be cleared and re-purposed without full re-instantiation, typically when the Role's configuration changes.

## Internal State & Concurrency

-   **State:** This class is highly stateful and mutable. Its core function is to maintain cached data. Key state includes `EntityList` objects for players and NPCs, `List` collections for items and markers, and several `Object2ByteMap` instances for caching Line of Sight checks. It also manages its own update cycle via TTL timers like `positionCacheNextUpdate` and `cacheTTL`.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be exclusively accessed and manipulated by its parent `Role` within the single-threaded server game loop. All internal collections and state variables are accessed without locks or other synchronization primitives.

    **WARNING:** Concurrent access from multiple threads will lead to `ConcurrentModificationException`, inconsistent AI perception, and unpredictable server behavior. All interactions with a PositionCache instance must be synchronized with the main server tick.

## API Surface

The public API is divided into configuration methods (called once during setup) and query methods (called frequently during AI updates).

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| reset(isConfiguring) | void | O(N) | Clears all internal lists and caches. Sets the configuration state. |
| finalizeConfiguration() | void | O(1) | Locks the cache, preventing further changes to distance requirements. |
| requirePlayerDistanceSorted(v) | void | O(1) | Registers a need to track players up to a given distance, sorted. Must be called during configuration. |
| tickPositionCacheNextUpdate(dt) | boolean | O(1) | Advances internal timers. Returns true if the main position cache requires an update. |
| getClosestPlayerInRange(...) | Ref | O(N) | Finds the closest player within a range matching a filter. N is the number of cached players. |
| hasLineOfSight(ref, targetRef, ...) | boolean | Varies | Performs a voxel raycast to check for obstructions. Results are cached aggressively. |
| isFriendlyBlockingLineOfSight(...) | boolean | Varies | Checks if a friendly NPC is intersecting a potential line of fire. |

## Integration Patterns

### Standard Usage

The PositionCache is not intended for direct use by general game systems. It is an internal component of a `Role`. A typical interaction is initiated by an AI behavior or sensor, which accesses the cache through its parent `Role` to gather information about the environment.

```java
// Within an AI Behavior's update method
Role role = this.getOwningRole();
PositionCache cache = role.getPositionCache();
ComponentAccessor<EntityStore> accessor = ...;

// Find the closest player within 16 blocks
Ref<EntityStore> target = cache.getClosestPlayerInRange(0, 16, accessor);

if (target != null) {
    // Check if we have a clear shot before acting
    if (cache.hasLineOfSight(role.getOwnerRef(), target, accessor)) {
        // Proceed with attack logic
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new PositionCache()`. The cache is intrinsically linked to a `Role` and must be managed by it. Manually creating a cache will result in a non-functional, disconnected object.
-   **Late Configuration:** Calling `require...` methods after `finalizeConfiguration` has been invoked will throw an `IllegalStateException`. All perception requirements must be declared during the initial setup phase.
-   **Ignoring Lifecycle Methods:** Failure to call `tickPositionCacheNextUpdate` and other update methods during the `Role`'s tick will result in a permanently stale cache, effectively blinding the NPC.
-   **Shared Caches:** Do not share a single PositionCache instance across multiple `Role` objects. The cache is tailored to a single NPC's position and perception needs.

## Data Pipeline

The PositionCache acts as a transformation layer, converting raw world state into actionable intelligence for an NPC.

> Flow:
> Server Tick -> Role Update -> **PositionCache Update Trigger (if TTL expired)** -> Query `EntityStore` for all entities in max range -> Populate internal `EntityList` buckets -> AI Behavior Logic -> **Query PositionCache API (e.g., getClosestPlayerInRange, hasLineOfSight)** -> AI Action

