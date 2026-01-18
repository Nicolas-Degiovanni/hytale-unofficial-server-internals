---
description: Architectural reference for AStarNodePoolProvider
---

# AStarNodePoolProvider

**Package:** com.hypixel.hytale.server.npc.navigation
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface AStarNodePoolProvider {
```

## Architecture & Concepts
The AStarNodePoolProvider interface defines a contract for systems that supply AStarNodePool instances. This component is a critical part of the server-side NPC pathfinding engine, specifically designed to manage memory allocation for A* search algorithm nodes.

Its primary architectural role is to decouple the pathfinding algorithm from the strategy of how node pools are managed and scoped. By abstracting the pool source, the engine can implement different pooling strategies (e.g., per-world, per-region, per-thread) without altering the core pathfinding logic. This is a classic **Strategy** or **Provider** pattern focused on performance optimization by minimizing object churn and garbage collection pressure during intensive pathfinding calculations.

An implementation of this interface acts as a centralized factory or registry for AStarNodePools, ensuring that pathfinding jobs receive an appropriate, and potentially isolated, memory pool to work with.

## Lifecycle & Ownership
As an interface, AStarNodePoolProvider itself has no lifecycle. The lifecycle described here applies to its concrete implementations.

- **Creation:** Implementations are expected to be instantiated once during the server's bootstrap sequence, likely by a world or navigation service manager. They are designed as long-lived, foundational services.
- **Scope:** An instance of an AStarNodePoolProvider implementation persists for the entire lifetime of the server process or the world it manages. It holds and manages the lifecycle of the underlying AStarNodePool objects.
- **Destruction:** The provider and all its managed pools are destroyed during server or world shutdown. Implementations should handle the clean release of all pooled memory.

## Internal State & Concurrency
- **State:** The interface is stateless. However, any concrete implementation is inherently **stateful**, as its core responsibility is to manage a collection of AStarNodePool objects. This state is mutable, as pools may be created on-demand.
- **Thread Safety:** **CRITICAL:** Implementations of this interface **must be thread-safe**. Pathfinding operations are frequently offloaded to worker threads to avoid blocking the main server tick. Multiple threads may request a node pool concurrently for different pathfinding jobs. Implementations must use appropriate synchronization mechanisms (e.g., concurrent collections, locks) to prevent race conditions when accessing or creating pools.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPool(int id) | AStarNodePool | O(1) / O(log N) | Retrieves a node pool associated with the given identifier. The identifier typically represents a world or dimension ID. Throws if the provider is not in a valid state. |

## Integration Patterns

### Standard Usage
The provider is injected into high-level navigation services. A pathfinding system retrieves a pool from the provider at the beginning of a path calculation and uses it for the duration of that single operation.

```java
// A pathfinding service retrieves the correct pool for a specific world
AStarNodePoolProvider poolProvider = context.getService(AStarNodePoolProvider.class);
AStarNodePool nodePool = poolProvider.getPool(world.getId());

// The pool is then used by the algorithm
Path path = AStarPathfinder.findPath(start, end, nodePool);
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Caching:** Do not retrieve a pool from the provider and cache it indefinitely within a pathfinding instance. Pools are managed by the provider and their lifecycle should not be interfered with. Always request the pool immediately before a pathfinding job.
- **Cross-World Usage:** Do not use a pool retrieved with one world ID for a pathfinding job in a different world. This can lead to memory leaks or state corruption if pools are scoped per-world.
- **Implementing Per-Entity:** Do not create a new implementation of this interface for every NPC or pathfinding request. Implementations are heavyweight, long-lived services.

## Data Pipeline
This component does not process a continuous stream of data. Instead, it acts as a resource provider in the pathfinding pipeline.

> Flow:
> Pathfinding Request -> Navigation Service -> **AStarNodePoolProvider.getPool()** -> AStarNodePool -> A* Algorithm Execution -> Path Result

