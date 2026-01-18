---
description: Architectural reference for DelayedSystem
---

# DelayedSystem

**Package:** com.hypixel.hytale.component.system
**Type:** Abstract System

## Definition
```java
// Signature
public abstract class DelayedSystem<ECS_TYPE> extends TickingSystem<ECS_TYPE> {
```

## Architecture & Concepts
The DelayedSystem is an abstract base class that provides a foundational pattern for creating systems that do not need to execute their logic on every single game tick. It extends TickingSystem, inheriting its integration with the core game loop, but introduces a time-accumulation mechanism to throttle its primary logic.

This class serves as a critical performance optimization tool. By allowing developers to specify an execution interval, it prevents expensive or low-frequency operations (e.g., periodic AI behavior checks, environmental effects, or resource regeneration) from consuming CPU resources on every frame.

Internally, it leverages the Entity Component System's resource management. Upon registration, it creates a private, system-specific resource, an instance of the inner class DelayedSystem.Data. This resource is responsible for a single task: accumulating the delta time between ticks. The core tick method, called by the engine, simply adds the frame's delta time to this resource. Only when the accumulated time exceeds the configured interval does the system fire its abstract delayedTick method, which contains the subclass's actual logic.

This design elegantly separates the timing mechanism from the system's logic, promoting clean, single-responsibility implementations.

## Lifecycle & Ownership
- **Creation:** A concrete implementation of DelayedSystem is instantiated by a higher-level manager, typically during world or session initialization. The execution interval must be provided via the constructor at this time. The system must then be registered with the appropriate SystemManager to be integrated into the game loop.
- **Scope:** An instance of a DelayedSystem persists for the entire lifetime of the context it is registered in, such as a game world or a client session. Its associated Data resource is owned by the ECS Store and shares the same scope.
- **Destruction:** The system is destroyed and its resources are reclaimed when the parent world or session is torn down. The ECS Store manages the cleanup of the associated DelayedSystem.Data resource, ensuring no memory leaks.

## Internal State & Concurrency
- **State:** The DelayedSystem class itself is effectively immutable after construction, holding only the final float intervalSec. All mutable state, specifically the time accumulator, is externalized to the private DelayedSystem.Data resource object, which is managed by the ECS Store. This design ensures that system state is handled consistently within the ECS framework.
- **Thread Safety:** This class is **not thread-safe** and is designed to be operated by a single game-loop thread. The tick method is expected to be called sequentially. Concurrent calls to tick from multiple threads will lead to race conditions on the time accumulator within the Data resource, resulting in unpredictable behavior and incorrect timing. All system logic should be confined to the engine's main update thread.

## API Surface
The public contract is minimal, primarily exposing the abstract method for implementation and the constructor for configuration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DelayedSystem(intervalSec) | constructor | O(1) | Constructs the system with a fixed execution interval in seconds. |
| tick(dt, systemIndex, store) | void | O(1) | Engine-invoked method. Accumulates delta time and triggers delayedTick when the interval is met. |
| delayedTick(fullDeltaTime, systemIndex, store) | abstract void | Varies | Abstract method to be implemented by subclasses. Contains the core logic that runs periodically. |

## Integration Patterns

### Standard Usage
A developer should extend DelayedSystem and implement the delayedTick method. The instance is then registered with the engine's system manager.

```java
// A system to spawn a particle effect every 5 seconds.
public class AmbientEffectSystem extends DelayedSystem<ClientECS> {

    public AmbientEffectSystem() {
        // Set the execution interval to 5.0 seconds.
        super(5.0f);
    }

    @Override
    public void delayedTick(float fullDeltaTime, int systemIndex, @Nonnull Store<ClientECS> store) {
        // This logic will only run approximately every 5 seconds.
        // The fullDeltaTime parameter will be >= 5.0f.
        spawnAmbientParticles();
    }
}

// Conceptual registration during client startup
SystemRegistry registry = client.getSystemRegistry();
registry.register(new AmbientEffectSystem());
```

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Never call the tick or delayedTick methods directly. They are lifecycle methods managed exclusively by the engine's system scheduler. Manual invocation will corrupt the time accumulator and break the system's contract.
- **Very Short Intervals:** Configuring a DelayedSystem with an interval smaller than a typical frame time (e.g., 0.01 seconds) is counter-productive. This negates the performance benefit and adds the overhead of the time-checking logic for no gain. For per-frame logic, extend TickingSystem directly.
- **State in Subclass Fields:** Avoid storing mutable state directly as fields in the subclass. If state is required, it should be managed through dedicated ECS components or resources to ensure it is handled correctly by the engine's state management and serialization systems.

## Data Pipeline
The primary data flowing through this system is time itself. The system acts as a temporal filter, transforming a high-frequency stream of delta-time updates into low-frequency, periodic logic execution events.

> Flow:
> Game Loop Tick (provides dt) -> TickingSystem Integration -> **DelayedSystem.tick** (accumulates dt into Data resource) -> Threshold Check -> **DelayedSystem.delayedTick** -> Subclass Logic Execution

