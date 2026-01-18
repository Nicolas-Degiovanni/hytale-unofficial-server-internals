---
description: Architectural reference for BlockRegionViewManager
---

# BlockRegionViewManager

**Package:** com.hypixel.hytale.server.npc.blackboard.view
**Type:** Service / Abstract Factory

## Definition
```java
// Signature
public abstract class BlockRegionViewManager<Type extends BlockRegionView<Type>> implements IBlackboardViewManager<Type> {
```

## Architecture & Concepts
The BlockRegionViewManager is an abstract base class that provides the core infrastructure for managing spatially-partitioned data views for the server-side AI system. It acts as a high-performance, thread-safe cache for `BlockRegionView` objects, which represent localized, queryable snapshots of world state used by NPCs for decision-making.

This class employs the **Template Method** design pattern. It defines the skeleton of the management algorithm—caching, retrieval, and cleanup—but defers the concrete implementation of view creation (`createView`) and cleanup logic (`shouldCleanup`) to subclasses. This allows for different types of specialized managers (e.g., a `PathfindingViewManager` or a `ThreatMapManager`) to share the same robust caching and lifecycle machinery while managing entirely different data.

The core architectural function is to abstract away the complexity of spatial partitioning. AI systems do not need to be aware of chunk boundaries or region indices; they can simply request a view for a given world position. The manager handles the translation of that position into a region index, and then lazily instantiates and returns the corresponding view using a get-or-create pattern.

### Lifecycle & Ownership
- **Creation:** An instance of a concrete subclass (e.g., `PathfindingViewManager`) is instantiated by the server's AI framework, typically once per world during its initialization phase. It is not intended for direct instantiation by game logic developers.
- **Scope:** The manager's lifecycle is bound to a server-side `Blackboard` instance, and by extension, to a specific world session. It persists as long as the world it manages is active.
- **Destruction:** The manager is eligible for garbage collection when its associated world is unloaded. The `clear` method is invoked to release all references to cached views, preventing memory leaks across world sessions. The `cleanup` method is part of its *internal* maintenance cycle, not its own destruction.

## Internal State & Concurrency
- **State:** The state is highly mutable. The primary internal state is the `views` map, a `Long2ObjectConcurrentHashMap` that caches `BlockRegionView` instances against their `long` region index. This cache grows and shrinks dynamically as NPCs traverse the world and views become active or stale.
- **Thread Safety:** This class is designed to be **thread-safe** for the most common operations. The use of `Long2ObjectConcurrentHashMap` ensures that concurrent calls to `get` from multiple NPC behavior threads can be handled safely without explicit locking.

    **WARNING:** While individual `get` and `put` operations are atomic, any compound operation that relies on a sequence of calls (e.g., check-then-act) is not. The `cleanup` process is specifically designed to be safe by first collecting indices to be removed into a separate `removalQueue` and then performing the removal in a second pass, avoiding a `ConcurrentModificationException`.

## API Surface
The public contract is focused on the retrieval and maintenance of views.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(long index, Blackboard blackboard) | Type | O(1) avg | The primary method. Retrieves a cached view or creates, caches, and returns a new one if it does not exist. |
| getIfExists(long index) | Type | O(1) avg | Retrieves a view only if it is already present in the cache. Returns null otherwise. |
| cleanup() | void | O(N) | Iterates all N cached views, determines which are stale via `shouldCleanup`, and removes them. **CRITICAL:** Must be called periodically to prevent memory leaks. |
| createView(long, Blackboard) | abstract Type | - | **Template Method.** Implemented by subclasses to define the instantiation logic for a specific view type. |
| shouldCleanup(Type) | abstract boolean | - | **Template Method.** Implemented by subclasses to define the conditions under which a view is considered stale and can be removed. |
| forEachView(Consumer) | void | O(N) | Executes a given action for each view in the cache. |

## Integration Patterns

### Standard Usage
A concrete implementation of this manager is retrieved from a central service or context. AI systems then use it to get a view for a specific location, which is then passed to AI behaviors.

```java
// Assume PathfindingViewManager extends BlockRegionViewManager
// and is registered with an AI service registry.

// 1. Retrieve the specific manager instance for the world's blackboard
PathfindingViewManager pathManager = blackboard.getViewManager(PathfindingViewManager.class);

// 2. Get the regional view for an NPC's current position
PathfindingView view = pathManager.get(npcTransform.getPosition(), blackboard);

// 3. Use the view to perform AI calculations
PathResult result = pathfinder.calculatePath(view, npc.getTarget());
```

### Anti-Patterns (Do NOT do this)
- **Neglecting Cleanup:** Failing to call the `cleanup` method on a regular basis (e.g., once every few server ticks) will lead to a severe memory leak. Stale views for regions that are no longer occupied by any entities will accumulate indefinitely.
- **Complex `shouldCleanup` Logic:** The `shouldCleanup` implementation should be lightweight. Performing expensive computations (e.g., database lookups, complex world queries) within this method will degrade server performance, as it is called for every cached view during the cleanup cycle.
- **Direct Instantiation:** This is an abstract class. Do not use `new BlockRegionViewManager()`. Concrete subclasses are managed by the AI framework and should be retrieved from the appropriate context or registry.

## Data Pipeline
The manager primarily serves as a cache in a request-response model rather than a linear data pipeline.

> Flow:
> AI Behavior Tick → Request View for NPC Position → **BlockRegionViewManager.get()** → Cache Hit? → [Yes] → Return Cached View
>
> ↘ [No]
>
> `createView()` → New View Instantiated → Store in Cache → Return New View
>
> ↘
>
> AI Behavior uses View Data for Decision Making

---
> Maintenance Cycle:
> Server Tick Scheduler → **BlockRegionViewManager.cleanup()** → `shouldCleanup()` check on all views → Stale Views Removed from Cache

