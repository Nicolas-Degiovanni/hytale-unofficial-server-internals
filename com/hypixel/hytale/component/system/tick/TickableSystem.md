---
description: Architectural reference for the TickableSystem interface, the core contract for game loop updates.
---

# TickableSystem

**Package:** com.hypixel.hytale.component.system.tick
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface TickableSystem<ECS_TYPE> extends ISystem<ECS_TYPE> {
    void tick(float deltaTime, int tickCount, @Nonnull Store<ECS_TYPE> componentStore);
}
```

## Architecture & Concepts

The TickableSystem interface is a fundamental contract within the Hytale Entity-Component-System (ECS) architecture. It defines the entry point for any system that requires continuous, time-based updates as part of the main game loop. This interface is the primary mechanism for evolving game state over time, such as moving entities, processing physics, or updating animations.

As a specialization of the base ISystem interface, TickableSystem mandates the implementation of a single method: `tick`. The engine's central `TickScheduler` is responsible for iterating through all registered TickableSystem implementations and invoking their `tick` method once per simulation step. This ensures a deterministic and ordered execution of game logic across the entire engine.

A system implementing this interface declares its intent to participate in the simulation's "heartbeat", receiving precise timing information and a direct handle to the component data it is responsible for managing.

### Lifecycle & Ownership

As an interface, TickableSystem does not have a lifecycle itself. The following applies to concrete classes that **implement** this interface.

-   **Creation:** Implementations are discovered and instantiated by a central `SystemRegistry` or a similar manager during world or session initialization. This process is typically automated through reflection or explicit registration in a configuration file.
-   **Scope:** An instance of a TickableSystem lives for the duration of the game world or client session it is associated with. It is created once and persists until the world is unloaded.
-   **Destruction:** Instances are marked for garbage collection when their parent `SystemRegistry` is destroyed, which occurs upon world shutdown or client exit.

## Internal State & Concurrency

-   **State:** Concrete implementations of TickableSystem are almost always stateful. They may cache entity queries, maintain internal data structures for performance, or track state across multiple ticks. The state is scoped to the system's specific domain, such as tracking all entities with a `PhysicsComponent`.

-   **Thread Safety:** Implementations are **not** expected to be thread-safe. The engine guarantees that all `tick` calls for a given world simulation occur sequentially on a single, dedicated game thread.

    **Warning:** It is a critical error to access or modify a TickableSystem instance from any thread other than the main game thread without explicit, robust synchronization. Doing so will lead to race conditions, data corruption, and engine instability.

## API Surface

The API surface consists of a single method that must be implemented.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(deltaTime, tickCount, store) | void | O(N) | Executes one simulation step for the system. Called by the engine's scheduler. The complexity is typically linear, proportional to the number of entities the system manages. |

## Integration Patterns

### Standard Usage

The standard pattern is to implement the interface, not to invoke it. A concrete class defines the logic that will be executed by the game loop.

```java
// A concrete implementation for handling entity movement.
public class MovementSystem implements TickableSystem<MovementComponent> {

    @Override
    public void tick(float deltaTime, int tickCount, @Nonnull Store<MovementComponent> store) {
        // Iterate through all entities with a MovementComponent
        for (Entity entity : store.getEntities()) {
            MovementComponent move = store.get(entity);
            PositionComponent pos = entity.getComponent(PositionComponent.class);

            // Apply velocity to position based on time passed
            pos.x += move.velocityX * deltaTime;
            pos.y += move.velocityY * deltaTime;
        }
    }
}

// This class would then be registered with the engine's SystemRegistry.
```

### Anti-Patterns (Do NOT do this)

-   **Manual Invocation:** Never call the `tick` method directly. The `TickScheduler` is solely responsible for its invocation. Calling it manually will break the execution order, disrupt delta time calculations, and bypass engine-level state management.
-   **Blocking Operations:** The `tick` method must execute quickly and be non-blocking. Performing file I/O, network requests, or heavy computations will freeze the game thread, causing severe performance degradation. Offload long-running tasks to asynchronous systems or worker threads.
-   **Cross-System Dependencies:** Avoid directly calling methods on another TickableSystem from within a `tick` implementation. This creates tight coupling and can lead to unpredictable behavior based on system execution order. Use the ECS `Store` or an event bus to communicate between systems.

## Data Pipeline

The `tick` method is a key processing stage in the main game loop. Its primary role is to read current state, apply logic, and write the new state back to the component store.

> Flow:
> Tick Scheduler -> **TickableSystem.tick(state)** -> Reads from Component Store -> Applies Game Logic -> Writes to Component Store -> Next System

