---
description: Architectural reference for SystemGroupDependency
---

# SystemGroupDependency

**Package:** com.hypixel.hytale.component.dependency
**Type:** Transient / Descriptor

## Definition
```java
// Signature
public class SystemGroupDependency<ECS_TYPE> extends Dependency<ECS_TYPE> {
```

## Architecture & Concepts
The SystemGroupDependency class is a declarative rule, not an active component. It serves as a fundamental building block for the Entity Component System's (ECS) scheduling mechanism. Its sole purpose is to define an execution order constraint between a single ISystem and an entire SystemGroup.

This object is consumed by the ComponentRegistry during the engine's startup or world-loading phase. It provides instructions to the DependencyGraph builder, allowing it to construct a directed acyclic graph (DAG) representing the final, ordered execution plan for all registered systems.

By depending on a logical group rather than concrete system classes, this mechanism promotes high cohesion and loose coupling. A developer can specify that a rendering system must run *after* all physics calculations without needing to explicitly list every physics-related system. This makes the architecture extensible and robust.

### Lifecycle & Ownership
- **Creation:** Instantiated declaratively by a developer inside an ISystem's `getDependencies` method. It is effectively metadata attached to a system.
- **Scope:** Short-lived and transient. An instance of SystemGroupDependency exists only during the dependency resolution phase when the ComponentRegistry is building its execution graph.
- **Destruction:** The object is eligible for garbage collection immediately after the DependencyGraph has been constructed. It holds no persistent state and is not referenced by any long-lived engine components.

## Internal State & Concurrency
- **State:** Immutable. The target SystemGroup, the ordering rule (BEFORE/AFTER), and the priority are set at construction via `final` fields and cannot be modified thereafter.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. However, it is designed to be used exclusively by the single thread responsible for initializing the ComponentRegistry and resolving the system execution order. Concurrent access is not an expected use case.

## API Surface
The public API is minimal and focused on internal consumption by the dependency resolution engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| validate(registry) | void | O(1) | Verifies that the target SystemGroup is registered. Throws IllegalArgumentException if the dependency is unsatisfiable. |
| resolveGraphEdge(registry, thisSystem, graph) | void | O(N) | The core logic. Iterates all N registered systems and adds directed edges to the graph to enforce the BEFORE or AFTER constraint. |
| getGroup() | SystemGroup | O(1) | Returns the target SystemGroup this dependency rule applies to. |

## Integration Patterns

### Standard Usage
The only correct way to use this class is to return it from within an ISystem's dependency declaration method. This informs the engine of the system's scheduling requirements.

```java
// Inside a custom ISystem implementation:
@Override
public Collection<Dependency<Game>> getDependencies() {
    return List.of(
        // Rule: This system must execute AFTER all systems
        // belonging to the CORE_PHYSICS group.
        new SystemGroupDependency<>(Order.AFTER, SystemGroups.CORE_PHYSICS),

        // Rule: This system must execute BEFORE all systems
        // belonging to the RENDERING group, with a high priority.
        new SystemGroupDependency<>(Order.BEFORE, SystemGroups.RENDERING, OrderPriority.HIGH)
    );
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Resolution:** Do not instantiate this class and attempt to call `resolveGraphEdge` manually. This method is strictly for internal use by the ComponentRegistry and DependencyGraph builders. Bypassing the registry will lead to an incomplete or incorrect execution order.
- **Stateful Dependencies:** Do not attempt to extend this class to add state or logic. A dependency is a static, declarative rule, not a dynamic object.
- **Circular Logic:** Creating dependencies that result in a logical cycle will cause a fatal error during engine startup. For example, System A declares it must run *after* the PHYSICS group, while a system within the PHYSICS group declares it must run *after* System A. The dependency resolver will detect this and halt execution.

## Data Pipeline
SystemGroupDependency acts as a configuration input into the system scheduling pipeline. It does not process game data itself.

> Flow:
> ISystem::getDependencies() -> **SystemGroupDependency Instance** -> ComponentRegistry -> DependencyGraph Builder -> Modified DependencyGraph -> Topological Sort -> Finalized System Execution Order

