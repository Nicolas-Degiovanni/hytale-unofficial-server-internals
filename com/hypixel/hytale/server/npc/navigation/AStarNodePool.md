---
description: Architectural reference for AStarNodePool
---

# AStarNodePool

**Package:** com.hypixel.hytale.server.npc.navigation
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface AStarNodePool {
```

## Architecture & Concepts
The AStarNodePool interface defines a contract for an object pool specialized in managing AStarNode instances. This component is a critical performance optimization within the server-side NPC navigation and pathfinding subsystem.

The A* pathfinding algorithm can generate an extremely high number of temporary node objects during a single search operation. Continuously allocating and deallocating these nodes on the heap would introduce significant garbage collector pressure, leading to performance degradation and server-wide stuttering.

This interface abstracts the pooling mechanism, allowing pathfinding algorithms to "rent" or "borrow" AStarNode objects instead of creating them with the **new** keyword. By recycling node objects, the system avoids the overhead of memory allocation and garbage collection, ensuring consistent and predictable performance for AI pathfinding, even under heavy load. Any class that implements this interface is expected to manage a collection of pre-allocated or recycled AStarNode objects.

## Lifecycle & Ownership
- **Creation:** Implementations of this interface are expected to be instantiated and managed by a higher-level service, such as a global NavigationService or a world-specific AI manager. The pool should be created once during server or world initialization.
- **Scope:** The pool's lifecycle is tied to the scope of the pathfinding system it serves. Typically, this means it persists for the entire duration of a game world's existence on the server.
- **Destruction:** The pool is destroyed when the world or server shuts down. There is no public contract for destruction; it is handled internally by the owning service.

**WARNING:** The ownership of the AStarNode objects themselves is temporarily transferred to the caller of the allocate method. The caller has a strict responsibility to return the object via deallocate once it is no longer needed. Failure to do so will result in a resource leak, eventually exhausting the pool and causing all subsequent pathfinding requests to fail.

## Internal State & Concurrency
- **State:** As an interface, AStarNodePool is stateless. However, any concrete implementation is inherently stateful, as it must maintain an internal collection (e.g., a list, queue, or stack) of available AStarNode objects.
- **Thread Safety:** Implementations of this interface **must be thread-safe**. NPC pathfinding can be a highly concurrent operation, with multiple agents across different threads requesting paths simultaneously. The underlying collection of nodes must be managed with appropriate synchronization mechanisms (e.g., locks, concurrent collections) to prevent race conditions during allocation and deallocation. Callers can assume that implementations are safe for concurrent access.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| allocate() | AStarNode | O(1) | Retrieves a pre-existing AStarNode from the pool. If the pool is empty, behavior is implementation-defined but may involve blocking or allocating a new object. |
| deallocate(AStarNode) | void | O(1) | Returns a previously allocated AStarNode to the pool, making it available for reuse. |

## Integration Patterns

### Standard Usage
The pool should be retrieved from a central service context. The standard pattern involves a try-finally block to guarantee that all allocated nodes are returned to the pool, even if the pathfinding logic throws an exception.

```java
// A pathfinding algorithm retrieves the pool from a service registry
AStarNodePool nodePool = context.getService(AStarNodePool.class);
List<AStarNode> usedNodes = new ArrayList<>();

try {
    // During the search, nodes are allocated as needed
    AStarNode startNode = nodePool.allocate();
    usedNodes.add(startNode);
    // ... perform pathfinding, allocating more nodes
} finally {
    // CRITICAL: Ensure all allocated nodes are returned
    for (AStarNode node : usedNodes) {
        nodePool.deallocate(node);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Node Leaking:** Failing to call deallocate for every node acquired via allocate. This is the most severe anti-pattern and will cripple the server's AI system over time by exhausting the pool.
- **Use-After-Free:** Continuing to hold a reference to or modify an AStarNode object after it has been passed to deallocate. That object's memory may be immediately re-allocated to another pathfinding operation on a different thread, leading to catastrophic state corruption.
- **Double Deallocation:** Calling deallocate more than once on the same AStarNode instance. This can corrupt the internal state of the pool, potentially leading to the same node being allocated to two different threads simultaneously.

## Data Pipeline
This component does not process a stream of data but rather manages a resource lifecycle.

> **Allocation Flow:**
> Pathfinding Algorithm (needs a node) -> Calls **allocate()** -> AStarNodePool (removes node from internal cache) -> Returns AStarNode instance to Algorithm

> **Deallocation Flow:**
> Pathfinding Algorithm (finished with node) -> Calls **deallocate(node)** -> AStarNodePool (resets node state and adds to internal cache) -> Node is now available for reuse

