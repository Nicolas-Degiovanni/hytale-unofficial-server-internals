---
description: Architectural reference for DependencyGraph
---

# DependencyGraph

**Package:** com.hypixel.hytale.component.dependency
**Type:** Transient

## Definition
```java
// Signature
public class DependencyGraph<ECS_TYPE> {
```

## Architecture & Concepts

The DependencyGraph is a foundational component of the Entity-Component-System (ECS) engine responsible for establishing a deterministic execution order for all registered game systems. In an ECS architecture, systems often have implicit or explicit dependencies on one another; for example, a PhysicsSystem must execute before a RenderingSystem to ensure the latest positions are drawn.

This class implements a topological sort algorithm to resolve these dependencies. It constructs a directed acyclic graph where each node is an ISystem and each directed edge represents a "must run before" relationship. The primary output is a linearly ordered array of systems that respects all declared dependencies.

This process is critical during the initialization phase of a game world or state. By guaranteeing a stable and correct execution order, the DependencyGraph prevents a wide class of bugs related to race conditions and out-of-order state updates within a single game tick. It acts as a short-lived "solver" or "planner" that is created, used once to determine the execution plan, and then discarded.

### Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level ECS orchestrator, such as a SystemManager or World, during its bootstrap sequence. It is provided with the complete, unordered set of systems that will be active.
- **Scope:** The object's lifetime is brief and confined to the system initialization phase. It is a single-use utility.
- **Destruction:** The DependencyGraph instance becomes eligible for garbage collection as soon as the `sort` method has returned and the resulting sorted array has been stored by the calling orchestrator. It holds no persistent state beyond this initial planning phase.

## Internal State & Concurrency
- **State:** The internal state is highly mutable. The class is designed to build a complex graph representation in memory through a series of method calls (`resolveEdges`, `addEdge`). This state includes maps of incoming and outgoing dependency edges for each system and a master sorted list of all edges. This state is not intended to be inspected or modified externally.

- **Thread Safety:** This class is **not thread-safe** and must only be used from a single thread. The internal data structures are manipulated without any synchronization primitives. Concurrent calls to `resolveEdges` or `sort` will result in a corrupted graph, leading to unpredictable system ordering, infinite loops, or runtime exceptions. This is by design, as system registration and sorting is a serialized, single-threaded bootstrap operation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DependencyGraph(systems) | constructor | O(N) | Constructs the graph with an initial set of N systems. Pre-allocates internal maps. |
| resolveEdges(registry) | void | O(N\*D) | Populates the graph by inspecting each of the N systems and their D dependencies. |
| sort(sortedSystems) | void | O(V\*E) | Performs a topological sort on the V systems and E edges. Throws IllegalArgumentException on cyclic dependencies. |
| addEdge(before, after, priority) | void | O(log E + E) | Manually inserts a dependency edge into the graph. Primarily used internally by `resolveEdges`. |

## Integration Patterns

### Standard Usage
The DependencyGraph must be used in a strict, sequential order to produce a valid result. The caller is responsible for orchestrating this lifecycle.

```java
// 1. Gather all systems that need to be sorted
ISystem[] allSystems = systemRegistry.getAllSystems();
ISystem[] sortedSystems = new ISystem[allSystems.length];

// 2. Create the graph solver
DependencyGraph graph = new DependencyGraph(allSystems);

// 3. Populate the graph with dependency information
graph.resolveEdges(componentRegistry);

// 4. Execute the sort to get the final execution order
try {
    graph.sort(sortedSystems);
} catch (IllegalArgumentException e) {
    // A cyclic dependency was detected. This is a critical configuration error.
    // The engine cannot proceed and must terminate.
    log.error("Fatal: Cyclic dependency detected in game systems!", e);
    throw new EngineInitializationException(e);
}

// The sortedSystems array now contains the correct execution plan for the game loop.
```

### Anti-Patterns (Do NOT do this)
- **Calling `sort` before `resolveEdges`:** The graph will be empty or incomplete, resulting in an incorrect or arbitrary system order. This will lead to subtle and difficult-to-diagnose bugs.
- **Reusing an Instance:** A DependencyGraph instance is stateful and single-use. Do not attempt to call `sort` multiple times or add new systems after sorting. For any change in the system set, a new DependencyGraph must be created.
- **Concurrent Access:** As stated in the concurrency section, accessing a DependencyGraph instance from multiple threads is strictly forbidden and will corrupt its internal state.

## Data Pipeline
The component acts as a processor that transforms an unordered collection of systems into a deterministically ordered execution plan based on dependency rules.

> Flow:
> Unordered `ISystem[]` -> **DependencyGraph** Constructor -> `resolveEdges()` reads metadata -> Internal Graph Structure is built -> `sort()` algorithm processes graph -> Sorted `ISystem[]` (Execution Plan)

