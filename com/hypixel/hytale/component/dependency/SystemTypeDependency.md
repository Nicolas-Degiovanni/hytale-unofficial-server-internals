---
description: Architectural reference for SystemTypeDependency
---

# SystemTypeDependency

**Package:** com.hypixel.hytale.component.dependency
**Type:** Transient

## Definition
```java
// Signature
public class SystemTypeDependency<ECS_TYPE, T extends ISystem<ECS_TYPE>> extends Dependency<ECS_TYPE> {
```

## Architecture & Concepts
The SystemTypeDependency class is a declarative rule used to establish execution order within the Entity-Component-System (ECS) framework. It represents a scheduling constraint, specifying that one system must execute either *before* or *after* all systems of a particular *type*.

This component is a cornerstone of the engine's dependency resolution mechanism. Instead of creating a rigid dependency on a specific system class instance, it defines a relationship with an abstract category of systems, identified by a SystemType. This abstraction is critical for engine modularity and extensibility, allowing new systems to be integrated into the execution pipeline correctly without modifying existing system definitions.

During the engine's initialization phase, these dependency objects are collected from all registered systems. They are then processed by a `DependencyGraph` builder, which translates these abstract rules into concrete directed edges in a graph representing the final execution order. The `resolveGraphEdge` method is the primary entry point where this translation occurs.

## Lifecycle & Ownership
- **Creation:** Instantiated declaratively by an `ISystem` implementer, typically within a static list or a factory method that defines the system's dependencies. It is considered part of a system's static metadata.
- **Scope:** Short-lived and ephemeral. Its sole purpose is to provide data to the dependency graph resolver during the engine's bootstrap sequence. It exists only during the graph construction phase.
- **Destruction:** The object becomes eligible for garbage collection immediately after the `DependencyGraph` has been fully resolved and the final, sorted execution order of systems is determined. No long-term references to SystemTypeDependency instances are maintained by the engine.

## Internal State & Concurrency
- **State:** **Immutable**. The core state, including the target `systemType`, the `order` (BEFORE/AFTER), and the `priority`, is established at construction and cannot be modified thereafter. The `systemType` field is marked as `final`.
- **Thread Safety:** **Thread-safe by immutability**. This object can be safely read from multiple threads. However, the process it participates in—the construction of the dependency graph via `resolveGraphEdge`—is a mutating operation on the graph and **must** be performed in a single-threaded context to prevent race conditions and ensure a deterministic execution order.

## API Surface
The public API is minimal, designed for consumption by the dependency resolution framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSystemType() | SystemType | O(1) | Returns the target system type for this dependency rule. |
| validate(registry) | void | O(1) | Verifies that the target SystemType is registered in the ComponentRegistry. Throws IllegalArgumentException if the dependency cannot be satisfied. |
| resolveGraphEdge(registry, thisSystem, graph) | void | O(N) | Adds one or more directed edges to the DependencyGraph. Complexity is linear to N, the total number of registered systems. |

**Warning:** The O(N) complexity of `resolveGraphEdge` implies that dependency resolution time scales with the total number of systems in the engine. This process is executed only at startup, but a very high system count could impact load times.

## Integration Patterns

### Standard Usage
A developer defines dependencies as part of a system's implementation. The framework consumes this metadata to build the execution graph.

```java
// Example of a system declaring a dependency
public class PlayerInputSystem implements ISystem<ClientEcs> {

    // The engine's dependency resolver will invoke this method
    @Override
    public List<Dependency<ClientEcs>> getDependencies() {
        // This rule ensures PlayerInputSystem runs AFTER all physics systems
        return List.of(
            new SystemTypeDependency<>(Order.AFTER, PhysicsSystem.TYPE)
        );
    }

    // ... other system methods
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Dependencies:** Do not attempt to modify a SystemTypeDependency object after creation or store state within it. They are intended to be immutable configuration objects.
- **Manual Resolution:** Never call `resolveGraphEdge` directly. This method is exclusively for the engine's internal dependency graph builder. Manual invocation will lead to an incomplete or corrupt execution graph.
- **Circular Logic:** Creating opposing dependencies will form a cycle in the graph, which will cause the dependency resolver to fail at startup. For example, System A declaring `BEFORE` System B's type, while a system of type B declares `AFTER` System A's type.

## Data Pipeline
SystemTypeDependency acts as a configuration input for the system scheduler. It does not process game data itself; rather, it dictates the flow of control between systems that do.

> Flow:
> System Class Definition -> **SystemTypeDependency Instance** -> DependencyGraph Resolver -> Directed Graph Edge -> Topologically Sorted Execution List

