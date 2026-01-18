---
description: Architectural reference for RemovalSystem
---

# RemovalSystem

**Package:** com.hypixel.hytale.builtin.instances.removal
**Type:** System Component

## Definition
```java
// Signature
public class RemovalSystem extends TickingSystem<ChunkStore> implements RunWhenPausedSystem<ChunkStore> {
```

## Architecture & Concepts
The RemovalSystem is a critical component of the server's instance management and resource lifecycle. It functions as an automated garbage collector for game worlds, specifically those designated as temporary or conditional instances (e.g., minigames, dungeons).

Operating within the server's Entity Component System (ECS) framework, this system is scheduled to run on every tick for the world it belongs to. Its sole responsibility is to evaluate a set of predefined criteria, or *Removal Conditions*, to determine if the world should be unloaded and removed from memory.

Key architectural points:
- **Decoupling:** It cleanly separates the *policy* of world removal (defined declaratively in `InstanceWorldConfig`) from the *mechanism* of world removal (executed by the `Universe`).
- **Asynchronous Execution:** The actual removal operation is dispatched to a background thread via `CompletableFuture`. This is a crucial design choice to prevent the world's main tick loop from stalling on potentially long-running I/O operations associated with unloading chunks and saving state.
- **Resilience:** By implementing `RunWhenPausedSystem`, it ensures that even idle or paused worlds are correctly evaluated for cleanup, preventing orphaned instances from consuming server resources indefinitely.

### Lifecycle & Ownership
- **Creation:** The RemovalSystem is not instantiated directly by developers. The ECS framework automatically creates an instance of this system for each `World` whose configuration specifies its inclusion.
- **Scope:** The lifecycle of a RemovalSystem instance is tightly bound to the `World` it monitors. It persists as long as the world exists within the `Universe`.
- **Destruction:** The system is destroyed and garbage collected as part of the `World` cleanup process when the world is finally removed from the `Universe`.

## Internal State & Concurrency
- **State:** The system itself is stateless. However, it uses a world-scoped `InstanceDataResource` to maintain a simple state machine. The `isRemoving` flag within this resource acts as a one-way latch to ensure the removal process is only triggered once.
- **Thread Safety:** This system is not thread-safe and is designed to be operated exclusively by its parent world's single-threaded tick loop.
    - The `tick` method is always invoked on the world's main thread.
    - The call to `Universe.removeWorld` is intentionally asynchronous. This offloads the blocking work from the game thread, but it also means that the world may continue to exist for several ticks after the removal process has been initiated. The `isRemoving` flag prevents re-triggering during this period.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, systemIndex, store) | void | O(N) | The main entry point called by the ECS scheduler. Evaluates conditions and triggers world removal. N is the number of configured RemovalConditions. |
| shouldRemoveWorld(store) | boolean | O(N) | A static helper that encapsulates the logic for evaluating all `RemovalCondition`s associated with the world. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly in code. Instead, they enable and configure it through the world's configuration files, typically by defining a set of `RemovalCondition`s.

**Example World Configuration (Conceptual YAML):**
```yaml
# In a world template file
systems:
  - com.hypixel.hytale.builtin.instances.removal.RemovalSystem

config:
  instanceWorldConfig:
    removalConditions:
      - type: NoPlayersForDuration
        duration: 300 # seconds
      - type: WorldLifetimeExceeded
        maxLifetime: 3600 # seconds
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new RemovalSystem()`. The ECS framework is solely responsible for its lifecycle. Manually creating an instance will result in a non-functional system that is not registered with any world.
- **Manual Ticking:** Do not call the `tick` method from user code. This bypasses the ECS scheduler and can lead to race conditions or incorrect state management, especially with the `isRemoving` flag.
- **Blocking Conditions:** When implementing a custom `RemovalCondition`, ensure its `shouldRemoveWorld` method is non-blocking and computationally inexpensive. A long-running condition will stall the entire world's tick loop, causing extreme server lag.

## Data Pipeline
The flow of data and control for a world removal event is linear and declarative.

> Flow:
> World Configuration File -> `InstanceWorldConfig` loaded by Server -> `RemovalCondition` array -> **RemovalSystem.shouldRemoveWorld()** evaluates conditions -> `boolean` result -> **RemovalSystem.tick()** checks result and `isRemoving` flag -> `CompletableFuture.runAsync(Universe::removeWorld)` is dispatched

