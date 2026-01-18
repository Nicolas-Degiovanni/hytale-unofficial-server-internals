---
description: Architectural reference for the ISystem interface, the core contract for all logic-processing units in the Entity Component System.
---

# ISystem

**Package:** com.hypixel.hytale.component.system
**Type:** Contract Interface

## Definition
```java
// Signature
public interface ISystem<ECS_TYPE> {
```

## Architecture & Concepts
The ISystem interface is the foundational contract for the "System" in an Entity Component System (ECS) architecture. It defines the behavioral blueprint for any class that processes component data each frame. Systems encapsulate game logic, such as physics, rendering, AI, or player input, keeping it separate from the data it operates on (Components) and the entities that own them.

The most critical architectural concept enforced by this interface is **dependency management**. Systems rarely operate in isolation; for example, a rendering system must execute *after* a physics system has updated entity positions. The `getDependencies` method provides the mechanism for a system to declare its prerequisites.

The static utility method `calculateOrder` consumes these dependency declarations to perform a topological sort on all registered systems. This produces a deterministic, stable execution order, which is paramount for preventing logical race conditions and ensuring predictable behavior within the game loop. The `ComponentRegistry` is the authority that manages the collection of systems and orchestrates this sorting process, typically once during initialization.

## Lifecycle & Ownership
As an interface, ISystem itself has no lifecycle. The following describes the lifecycle of a concrete class that **implements** ISystem.

-   **Creation:** System instances are created during the application's bootstrap or world-loading phase. They are designed to be instantiated by a central authority, such as a `ClientEntryPoint` or `WorldInitializer`, and immediately registered with the active `ComponentRegistry`.
-   **Scope:** A system's lifetime is bound to the `ComponentRegistry` that owns it. Typically, this means a system persists for the entire duration of a game session, world, or server instance.
-   **Destruction:** When the owning `ComponentRegistry` is shut down (e.g., on world unload or client exit), it will iterate through its registered systems and invoke the `onSystemUnregistered` callback. After this point, the system instance is dereferenced and becomes eligible for garbage collection.

## Internal State & Concurrency
-   **State:** The interface itself is stateless. However, concrete implementations are almost always stateful. They may cache entity queries, hold references to subsystems, or manage internal data structures to optimize their processing logic.
-   **Thread Safety:** This interface makes no guarantees about thread safety. The execution of systems is typically managed by a single, main game-loop thread. Therefore, implementations are not required to be thread-safe unless they explicitly interact with asynchronous services like networking or file I/O.

    **Warning:** Accessing a system from a thread other than the main game thread is inherently unsafe and will lead to concurrency violations unless the system is explicitly designed with locks or other synchronization primitives.

## API Surface
The API surface of ISystem is primarily composed of lifecycle callbacks and metadata methods used by the `ComponentRegistry`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onSystemRegistered() | void | O(1) | Callback invoked by the ComponentRegistry immediately after the system is added. |
| onSystemUnregistered() | void | O(1) | Callback invoked by the ComponentRegistry just before the system is removed. |
| getGroup() | SystemGroup | O(1) | Returns the group this system belongs to, used for coarse-grained organization. |
| getDependencies() | Set | O(1) | **Crucial.** Returns the set of dependencies required for this system to function correctly. |
| calculateOrder(registry, systems, size) | static void | O(V+E) | Sorts the provided array of systems in-place based on their dependency graph. |

## Integration Patterns

### Standard Usage
The primary pattern is to implement this interface to create a new piece of game logic. The implementation is then registered with the engine's central `ComponentRegistry`.

```java
// 1. Define a concrete system implementing the interface
public class PhysicsSystem implements ISystem<GameEcs> {
    @Override
    public Set<Dependency<GameEcs>> getDependencies() {
        // This system depends on the existence of a TransformComponent
        return Set.of(Dependency.component(TransformComponent.class));
    }

    // ... other update logic
}

// 2. Register the system during initialization
ComponentRegistry<GameEcs> registry = getRegistry();
registry.registerSystem(new PhysicsSystem());

// 3. The engine later calculates the order for all systems
// This is typically not called by user code.
ISystem.calculateOrder(registry, allSystems, systemCount);
```

### Anti-Patterns (Do NOT do this)
-   **Manual Sorting:** Do not call `ISystem.calculateOrder` directly. This is an internal utility for the `ComponentRegistry` to manage the global execution order. Manually invoking it can disrupt the engine's state.
-   **Omitting Dependencies:** Failing to declare a dependency is a severe architectural error. If `RenderSystem` depends on `PhysicsSystem` but does not declare it, the engine may execute them in the wrong order, causing single-frame flickering, stale data, and other difficult-to-debug logical errors.
-   **Dynamic Dependencies:** The set returned by `getDependencies` should be constant. Computing it dynamically or returning a different set on subsequent calls is an anti-pattern that will break the stability of the dependency graph sort.

## Data Pipeline
ISystem is a processing stage within the main game loop's data pipeline. It does not handle data ingress or egress itself but acts upon data managed by the `ComponentRegistry`.

> Flow:
> Engine Bootstrap -> `ComponentRegistry` Instantiation -> **System Implementation Registration** -> Dependency Graph Resolution via `calculateOrder` -> Game Loop Tick -> Iterate and Execute Sorted Systems -> State Mutation in Components

