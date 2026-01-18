---
description: Architectural reference for Dependency
---

# Dependency

**Package:** com.hypixel.hytale.component.dependency
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class Dependency<ECS_TYPE> {
```

## Architecture & Concepts
The Dependency class is an abstract base for defining execution-order constraints between different ISystem implementations within the Hytale Entity-Component-System (ECS) framework. It is a foundational component of the engine's startup and system scheduling logic.

Instances of Dependency subclasses do not represent systems themselves; rather, they are declarative metadata objects that specify relationships. For example, a Dependency can declare that *PhysicsSystem* must execute *before* *RenderingSystem*.

During the engine's initialization phase, the ComponentRegistry collects all Dependency objects declared by registered systems. It then feeds these constraints into a DependencyGraph, which performs a topological sort to establish a deterministic, valid execution order for the main game loop. This mechanism prevents race conditions and ensures that systems which rely on the output of other systems are always processed in the correct sequence.

The generic parameter ECS_TYPE allows this dependency framework to be reused for different ECS worlds, such as the client and server, which may have distinct sets of components and systems.

## Lifecycle & Ownership
- **Creation:** Instances of concrete Dependency subclasses (e.g., BeforeDependency, AfterDependency) are instantiated directly by ISystem implementations, typically as part of their definition or within a dedicated dependency declaration method. They represent a core part of a system's contract with the engine.
- **Scope:** These objects are ephemeral and exist only during the dependency resolution phase of engine initialization. They are configuration data, not long-lived runtime objects.
- **Destruction:** Once the DependencyGraph has been successfully built and the final system execution order is cached, the Dependency objects are no longer referenced by the core engine and are eligible for garbage collection. Their lifecycle is tied exclusively to the application's bootstrap sequence.

## Internal State & Concurrency
- **State:** The internal state, consisting of an Order and a priority, is immutable. It is set once via the constructor and cannot be changed, which is critical for predictable and repeatable dependency resolution.
- **Thread Safety:** The class is inherently thread-safe due to its immutable state. However, the dependency resolution process in which it participates is a strictly single-threaded operation that occurs during engine startup. Modifying the ComponentRegistry or DependencyGraph from multiple threads during this phase is unsupported and would lead to catastrophic engine failure.

## API Surface
The public API is designed for consumption by the dependency resolution framework, not by typical game logic developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getOrder() | Order | O(1) | Returns the type of ordering constraint (e.g., BEFORE, AFTER). |
| getPriority() | int | O(1) | Returns the priority of the constraint, used for resolving ties. |
| validate(ComponentRegistry) | void | O(log N) | **Abstract.** Verifies that the dependency is valid within the given registry (e.g., checks if the target system exists). Throws an exception on failure. |
| resolveGraphEdge(ComponentRegistry, ISystem, DependencyGraph) | void | O(log V) | **Abstract.** Instructs the DependencyGraph to add a directed edge representing this constraint. V is the number of vertices (systems) in the graph. |

## Integration Patterns

### Standard Usage
A developer implementing a new system declares dependencies by creating and returning them. The engine's framework consumes these objects; the developer does not invoke methods on them.

```java
// Inside a custom ISystem implementation
@Override
public List<Dependency<ClientECS>> getDependencies() {
    // This system must run AFTER the PlayerInputSystem
    return List.of(new AfterDependency<>(PlayerInputSystem.class));
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Do not call `validate` or `resolveGraphEdge` yourself. These methods are internal to the engine's bootstrap process and are managed exclusively by the ComponentRegistry. Manual invocation will corrupt the dependency graph.
- **Circular Dependencies:** Defining dependencies that create a cycle (e.g., System A runs before B, and System B runs before A) is a critical design flaw. The dependency graph resolver is designed to detect this and will terminate the application with an error.
- **Stateful Dependencies:** Do not create subclasses of Dependency that contain mutable state. The resolution process assumes these objects are immutable configuration data.

## Data Pipeline
This class does not process runtime game data. Instead, it defines the structure of the system processing pipeline itself.

> Flow:
> ISystem Class Definition -> **Dependency Object Instantiation** -> ComponentRegistry Collection -> DependencyGraph Construction -> Sorted System Execution Order

