---
description: Architectural reference for SystemDependency
---

# SystemDependency

**Package:** com.hypixel.hytale.component.dependency
**Type:** Transient

## Definition
```java
// Signature
public class SystemDependency<ECS_TYPE, T extends ISystem<ECS_TYPE>> extends Dependency<ECS_TYPE> {
```

## Architecture & Concepts
SystemDependency is a declarative metadata object used to define the execution order of systems within the Entity Component System (ECS) framework. It is not a long-lived service but rather a configuration primitive used during the engine's bootstrap phase.

Its fundamental purpose is to express a relationship between two systems, such as "System A must execute *before* System B". These declarations are collected by the ComponentRegistry and fed into a DependencyGraph. The graph is then topologically sorted to produce a deterministic, linear execution order for all registered systems for each game tick.

This class is a critical component of the engine's startup sequence, ensuring that systems that produce data (e.g., InputSystem) are executed before systems that consume that data (e.g., PlayerMovementSystem). It enforces a predictable and manageable update loop.

### Lifecycle & Ownership
- **Creation:** Instantiated directly by developers when implementing a new ISystem. A system will typically create and return a list of its dependencies from a dedicated method.
- **Scope:** Transient and short-lived. A SystemDependency object exists only during the system registration and dependency resolution phase managed by the ComponentRegistry.
- **Destruction:** The object is eligible for garbage collection immediately after the DependencyGraph has been constructed. It holds no state that persists beyond engine initialization.

## Internal State & Concurrency
- **State:** Immutable. The target system class, order (BEFORE/AFTER), and priority are set via the constructor and cannot be modified. This immutability guarantees predictable behavior during graph construction.
- **Thread Safety:** The class itself is thread-safe due to its immutable state. However, the process it participates in—the construction of the DependencyGraph—is **not** thread-safe. The entire system registration and dependency resolution process must be performed in a single-threaded context to prevent race conditions and ensure a consistent system execution order.

## API Surface
The public API is minimal, designed exclusively for consumption by the ComponentRegistry and DependencyGraph.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSystemClass() | Class<T> | O(1) | Returns the class of the system this dependency targets. |
| validate(registry) | void | O(1) | Confirms that the target system is registered. Throws IllegalArgumentException if the dependency cannot be satisfied. |
| resolveGraphEdge(registry, thisSystem, graph) | void | O(N) | Adds a directed edge to the DependencyGraph representing the ordering constraint. N is the total number of registered systems. |

## Integration Patterns

### Standard Usage
A developer implementing a custom system declares its dependencies to the engine. The engine's ComponentRegistry consumes these objects during its initialization sequence.

```java
// Inside a custom ISystem implementation:
@Override
public List<Dependency<ECS_TYPE>> getDependencies() {
    List<Dependency<ECS_TYPE>> dependencies = new ArrayList<>();
    // Declare that this system must run AFTER the PhysicsSystem.
    dependencies.add(new SystemDependency<>(Order.AFTER, PhysicsSystem.class));
    return dependencies;
}
```

### Anti-Patterns (Do NOT do this)
- **Circular Dependencies:** Declaring dependencies that form a cycle (e.g., A runs before B, and B runs before A) will cause the dependency graph resolution to fail, preventing the engine from starting.
- **State Transfer:** Do not attempt to use this class or its subclasses to pass data or state between systems. Its sole purpose is to define execution order. Systems should communicate via shared components on entities or through a dedicated event bus.

## Data Pipeline
SystemDependency acts as a node in a control-flow pipeline, not a data pipeline. It transforms a developer's declaration into a structural element of the engine's main loop.

> Flow:
> ISystem Definition -> **SystemDependency Instance** -> ComponentRegistry Collection -> DependencyGraph.addEdge() -> Topologically Sorted Execution List

