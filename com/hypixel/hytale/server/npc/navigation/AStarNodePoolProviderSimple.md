---
description: Architectural reference for AStarNodePoolProviderSimple
---

# AStarNodePoolProviderSimple

**Package:** com.hypixel.hytale.server.npc.navigation
**Type:** Resource (Factory/Provider)

## Definition
```java
// Signature
public class AStarNodePoolProviderSimple implements AStarNodePoolProvider, Resource<EntityStore> {
```

## Architecture & Concepts
The AStarNodePoolProviderSimple is a foundational component of the server-side NPC pathfinding system. Its sole responsibility is to manage and dispense memory-efficient pools of A* nodes (AStarNodePool).

This class implements a **Flyweight design pattern** to drastically reduce object allocation and garbage collection overhead during intensive pathfinding calculations. Instead of creating a new node pool for every pathfinding request, it maintains an internal, lazily-populated cache of pools. Each pool is uniquely identified by its capacity, specifically the maximum number of child nodes a single node can have.

Architecturally, this provider is designed as a **Resource** scoped to a specific **EntityStore**. This is a critical design choice ensuring that pathfinding resources for one world or dimension are completely isolated from others. It acts as a stateful factory, but its state is confined within the context of its parent EntityStore.

## Lifecycle & Ownership
- **Creation:** An instance is not created directly. The Hytale engine's resource management system instantiates this class via its `clone()` method when an EntityStore is initialized. It is requested and attached to the EntityStore using its unique ResourceType handle, retrieved from `NPCPlugin`.
- **Scope:** The lifecycle of an AStarNodePoolProviderSimple instance is strictly bound to the lifecycle of its parent EntityStore. It persists in memory for as long as the world or dimension it belongs to is active.
- **Destruction:** The object and its entire cache of node pools are de-referenced and become eligible for garbage collection when the associated EntityStore is unloaded. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** This class maintains a mutable, lazily-populated state in the `nodePools` map. This map grows on-demand as pathfinding jobs request pools with different `childNodeCount` configurations. The state is entirely in-memory and is not persisted.
- **Thread Safety:** This implementation is **not thread-safe**. The underlying `Int2ObjectOpenHashMap` is not synchronized. Concurrent invocations of `getPool` from multiple threads can lead to race conditions within the `computeIfAbsent` operation, resulting in map corruption or unpredictable behavior.

> **Warning:** All access to a given AStarNodePoolProviderSimple instance must be externally synchronized. In the Hytale server architecture, this is typically handled by a world-tick-based job scheduler that processes pathfinding requests serially for a given EntityStore, thus preventing concurrent access.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPool(int childNodeCount) | AStarNodePool | Amortized O(1) | Retrieves a cached, or creates a new, AStarNodePool configured for the specified child node count. This is the primary factory method. |

## Integration Patterns

### Standard Usage
This class is a low-level infrastructure component and should not be invoked directly by high-level game logic. It is consumed by intermediary pathfinding services which manage the overall pathfinding process for an EntityStore.

```java
// Hypothetical usage within a higher-level pathfinding service

// 1. Obtain the provider from the relevant EntityStore's resource context
EntityStore worldEntityStore = getCurrentWorld().getEntityStore();
AStarNodePoolProviderSimple provider = worldEntityStore.getResource(AStarNodePoolProviderSimple.getResourceType());

// 2. Request a pool suitable for a grid where nodes have up to 8 neighbors
AStarNodePool nodePool = provider.getPool(8);

// 3. Pass the acquired pool to the core A* algorithm for its execution
PathResult result = aStarAlgorithm.findPath(startPos, endPos, nodePool);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new AStarNodePoolProviderSimple()`. The engine's resource system is solely responsible for the lifecycle of this object. Direct instantiation breaks the `EntityStore` scoping and will lead to memory leaks.
- **Cross-World Sharing:** Do not cache and reuse a provider instance across different `EntityStore` contexts. This violates the architectural principle of world resource isolation.
- **Unsynchronized Concurrent Access:** Do not call `getPool` from multiple threads without an external locking mechanism. This will corrupt the internal state of the provider.

## Data Pipeline
This component does not process a stream of data. Instead, it acts as a resource factory within a larger request-response flow.

> **Object Provisioning Flow:**
> Pathfinding Job Request → Pathfinder Service → `EntityStore.getResource()` → **AStarNodePoolProviderSimple.getPool()** → AStarNodePool → A* Algorithm Execution

