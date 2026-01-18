---
description: Architectural reference for SteppableTickingSystem
---

# SteppableTickingSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** System Base Class

## Definition
```java
// Signature
public abstract class SteppableTickingSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
SteppableTickingSystem is an abstract base class within the server-side Entity Component System (ECS) framework. It serves as a specialized variant of EntityTickingSystem, designed to provide a conditional time-stepping mechanism for entities, primarily Non-Player Characters (NPCs).

Its core architectural function is to intercept the standard per-entity tick and modify the flow of time based on the entity's state. It decouples an entity's logical update from the server's main clock, enabling behaviors like pausing, single-step debugging, or scripted, frame-by-frame animations.

The system operates on a simple conditional principle:
1.  It first checks if an entity is considered **frozen**. An entity is frozen if it possesses a Frozen component or if the global world configuration flag isAllNPCFrozen is enabled.
2.  If the entity is **not frozen**, the system behaves like a standard EntityTickingSystem, passing the real frame delta time (dt) to the concrete implementation's steppedTick method.
3.  If the entity **is frozen**, the system checks for the presence of a StepComponent. If this component exists, the system uses the time value from StepComponent instead of the real delta time. This allows external systems to command a frozen entity to advance its logic by a specific, discrete amount of time, effectively "stepping" it forward one tick. If no StepComponent is present, the tick is aborted for that entity.

This class is a foundational building block for any server-side AI or entity logic that needs to be controlled independently of the main game loop's progression.

## Lifecycle & Ownership
-   **Creation:** Concrete subclasses of SteppableTickingSystem are instantiated by the server's ECS SystemGraph during the world initialization phase. They are discovered and registered as part of the server bootstrap process.
-   **Scope:** A single instance of a given system persists for the entire lifetime of the server world. These systems are long-lived, stateless processors.
-   **Destruction:** Instances are discarded and eligible for garbage collection only when the server world is shut down and the managing SystemGraph is torn down.

## Internal State & Concurrency
-   **State:** This class is stateless. The fields stepComponentType and frozenComponentType are immutable, final references to component definitions, initialized at construction. All operational data is passed as arguments to the tick method, ensuring that the system itself does not retain any per-entity or per-tick state.
-   **Thread Safety:** **This class is not thread-safe and must only be accessed by the main server thread.** Like most ECS systems, it is designed to be executed within a single-threaded game loop. All state mutations must be performed through the provided CommandBuffer, which queues changes to be applied at a safe synchronization point later in the tick cycle. Unsynchronized access will lead to data corruption and undefined behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, buffer) | void | O(1) | Overrides the parent method. Contains the core logic for determining the effective delta time and invoking steppedTick. Called by the ECS scheduler once per entity per frame. |
| steppedTick(tickLength, index, chunk, store, buffer) | void | *Abstract* | The primary extension point for subclasses. This method contains the actual entity logic and receives either the real delta time or a stepped time value. |

## Integration Patterns

### Standard Usage
A developer should extend this class to implement specific entity logic that requires pausable or steppable behavior. The system is then registered with the server's ECS framework, which will automatically invoke it.

```java
// Example of a concrete implementation for NPC pathfinding
public class NpcPathfindingSystem extends SteppableTickingSystem {
    @Override
    public void steppedTick(
        float dt,
        int index,
        @Nonnull ArchetypeChunk<EntityStore> archetypeChunk,
        @Nonnull Store<EntityStore> store,
        @Nonnull CommandBuffer<EntityStore> commandBuffer
    ) {
        // Implement pathfinding logic here using the provided dt.
        // This logic will now automatically respect the Frozen and Step components.
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new MySystem()`. The ECS SystemGraph is solely responsible for the lifecycle of all systems. Manually created instances will not be registered or ticked.
-   **Manual Invocation:** Do not call the `tick` or `steppedTick` methods directly. This bypasses the ECS scheduler and its safety mechanisms, such as the CommandBuffer, leading to severe concurrency issues and world state corruption.
-   **Storing State in Fields:** Subclasses must not store per-entity or per-tick data in instance fields. This violates the stateless principle of ECS systems and will cause incorrect behavior, as a single system instance processes thousands of entities. All state must be stored in components.

## Data Pipeline
The data flow for this system is driven by the main server game loop and the ECS scheduler.

> Flow:
> Server Game Loop -> ECS Scheduler -> **SteppableTickingSystem.tick(dt)** -> (Time Delta Calculation) -> **Subclass.steppedTick(effective_dt)** -> CommandBuffer -> World State Update

