---
description: Architectural reference for TimerSystem
---

# TimerSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** System (ECS)

## Definition
```java
// Signature
public class TimerSystem extends SteppableTickingSystem {
```

## Architecture & Concepts
The TimerSystem is a foundational component of the server-side Entity Component System (ECS). Its sole responsibility is to process and advance the state of time-based logic attached to entities. It operates exclusively on entities that possess a **Timers** component.

Architecturally, this class embodies the "S" in SOLID principles (Single Responsibility). It does not contain any game logic itself; rather, it acts as a high-performance engine for iterating over `Tickable` objects owned by other components. By extending SteppableTickingSystem, it signals its integration into the server's main tick loop and its capability for parallel execution, making it highly efficient for managing timers across thousands of entities simultaneously.

The system is data-driven. Its behavior is entirely determined by the data within the **Timers** components it queries. It does not create, destroy, or modify entities or components directly, ensuring predictable and isolated execution.

### Lifecycle & Ownership
-   **Creation:** The TimerSystem is instantiated once by the server's central ECS System Manager during the bootstrap sequence. Its dependencies, such as the specific component type it targets, are injected at construction. It is never created or managed by gameplay code.
-   **Scope:** The instance is a long-lived singleton that persists for the entire server session. It is registered with the ECS scheduler and participates in every server tick.
-   **Destruction:** The system is destroyed and its resources are released only during a full server shutdown, as part of the ECS world teardown process.

## Internal State & Concurrency
-   **State:** The TimerSystem is **stateless**. Its internal fields are immutable references configured at creation time. It does not cache or store any data related to the entities it processes between ticks. All state is read from and written to the **Timers** components in the main entity store.

-   **Thread Safety:** This system is designed for parallel execution and is considered thread-safe *within the context of the ECS scheduler*. The `isParallel` method allows the scheduler to partition the set of matching entities and invoke `steppedTick` for different entities across multiple worker threads.

    **WARNING:** The `steppedTick` method is not re-entrant and must not be called manually from arbitrary threads. Concurrency is managed exclusively by the parent ECS scheduler, which guarantees that each thread operates on a distinct and non-overlapping set of entity data.

## API Surface
The public API is primarily for consumption by the ECS framework, not for direct use by developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| steppedTick(dt, index, archetypeChunk, store, commandBuffer) | void | O(N) | Processes a single entity. Ticks all `Tickable` objects within its **Timers** component. N is the number of timers on the component. |
| getQuery() | Query | O(1) | Framework contract. Returns the query for entities with a **Timers** component. |
| getDependencies() | Set | O(1) | Framework contract. Declares dependencies to ensure correct execution order by the scheduler. |

## Integration Patterns

### Standard Usage
A developer does not interact with the TimerSystem directly. To use its functionality, one must add a **Timers** component to an entity. The system will automatically discover and process this component on each server tick.

```java
// This logic would exist within another system responsible for entity setup.
// The CommandBuffer ensures thread-safe component addition.

// 1. Create a component to hold one or more Tickable objects.
Timers timersComponent = new Timers();

// 2. Add a custom timer implementation.
// This example might be a cooldown timer for an ability.
timersComponent.add(new CooldownTimer(5.0f)); // 5-second timer

// 3. Add the component to the target entity.
commandBuffer.addComponent(targetEntity, timersComponent);

// The TimerSystem will now automatically call tick() on CooldownTimer every frame.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new TimerSystem()`. The ECS framework is solely responsible for its lifecycle. Attempting to create a rogue instance will result in a non-functional system that is not registered with the game loop.
-   **Manual Invocation:** Do not call `steppedTick` directly. This bypasses the scheduler, dependency resolution, and parallel processing, which can lead to race conditions, corrupted state, and severe performance degradation.
-   **Stateful Tickables:** While the system itself is safe, the `Tickable` objects it processes can introduce concurrency issues. If a `Tickable` object accesses shared, mutable state from another component or global system, it must be implemented with thread safety in mind, as the TimerSystem may process it on any worker thread.

## Data Pipeline
The TimerSystem acts as a processor in a simple, linear data flow orchestrated by the ECS scheduler.

> Flow:
> Server Tick Start -> ECS Scheduler -> **TimerSystem.steppedTick()** -> Reads `Timers` Component from Entity Store -> Invokes `Tickable.tick(dt)` -> Mutates state within the `Tickable` object.

