---
description: Architectural reference for AStarNodePoolSimple
---

# AStarNodePoolSimple

**Package:** com.hypixel.hytale.server.npc.navigation
**Type:** Object Pool

## Definition
```java
// Signature
public class AStarNodePoolSimple implements AStarNodePool {
```

## Architecture & Concepts
AStarNodePoolSimple is a performance-critical component that implements the **Object Pool** design pattern. Its sole function is to manage the lifecycle of AStarNode objects, minimizing the overhead of memory allocation and garbage collection during intensive pathfinding calculations.

In the context of the A* search algorithm, thousands of nodes can be created and discarded per second. Repeatedly instantiating new objects places significant pressure on the Java Garbage Collector (GC), potentially leading to performance stutters or pauses on the server. This class mitigates that pressure by recycling AStarNode instances. Instead of creating a new object, the pathfinding system can "rent" one from the pool via the allocate method and "return" it via the deallocate method.

This implementation uses the fastutil ObjectArrayList for its internal storage, which is optimized for primitive and object collections, further underscoring its role as a low-level performance utility within the NPC navigation system.

### Lifecycle & Ownership
- **Creation:** An AStarNodePoolSimple instance is created by a higher-level pathfinding coordinator or executor at the beginning of a specific pathfinding task. It is not a global singleton.
- **Scope:** The object's lifetime is intentionally brief, scoped exclusively to a single, complete pathfinding operation. A new pool is instantiated for each path request.
- **Destruction:** The instance is abandoned after the pathfinding result is computed. It, along with all the AStarNode objects it contains, becomes eligible for garbage collection. There is no explicit cleanup or close method.

## Internal State & Concurrency
- **State:** The internal state is highly mutable. The core component is the nodePool list, which is continuously modified as nodes are allocated and deallocated. The state represents the cache of available, reusable AStarNode objects.

- **Thread Safety:** **This class is not thread-safe.** The underlying ObjectArrayList is not a synchronized collection, and no locking mechanisms are employed. It is designed with the strict assumption that it will be confined to and accessed by a single thread for the duration of one pathfinding task.

    **Warning:** Sharing an instance of AStarNodePoolSimple across multiple threads will result in race conditions, state corruption, and catastrophic pathfinding failures. Each pathfinding job running on a separate thread must have its own dedicated pool instance.

## API Surface
The public contract is minimal, focusing exclusively on the allocation and deallocation of AStarNode objects.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| allocate() | AStarNode | O(1) | Retrieves a recycled node from the pool. If the pool is empty, a new AStarNode is instantiated. |
| deallocate(AStarNode node) | void | O(1) | Returns a used AStarNode to the pool, making it available for future allocation requests. |

## Integration Patterns

### Standard Usage
The pool is intended to be used by a pathfinding algorithm to acquire and release nodes as it explores the search space. The typical pattern involves creating the pool at the start of the job and using it throughout the algorithm's execution.

```java
// A pathfinding executor creates a pool for a single job
AStarNodePoolSimple nodePool = new AStarNodePoolSimple(MAX_NODE_CHILDREN);
PathfindingAlgorithm search = new PathfindingAlgorithm(world, nodePool);

// Inside the algorithm...
AStarNode nextNode = nodePool.allocate();
// ... use the node for calculations

// When a node is no longer needed, it is returned
nodePool.deallocate(nextNode);
```

### Anti-Patterns (Do NOT do this)
- **Instance Sharing:** Never share a single pool instance across multiple pathfinding jobs that could run concurrently. Each job requires its own isolated pool.
- **Node Leaking:** Forgetting to call deallocate on a node after use negates the benefit of the pool. While the Java GC will eventually collect the object, the pool cannot recycle it, leading to degraded performance as new objects must be created instead.
- **External Deallocation:** Do not return a node to a pool it was not allocated from. This can lead to cross-contamination of state between different pathfinding jobs.

## Data Pipeline
This component does not process data in a traditional pipeline. Instead, it manages a memory lifecycle.

> Flow:
> Pathfinding Algorithm requires a node -> **AStarNodePoolSimple.allocate()** -> A recycled or new AStarNode is returned -> Algorithm uses the node -> Algorithm discards the node -> **AStarNodePoolSimple.deallocate()** -> Node is stored in the internal pool for reuse.

