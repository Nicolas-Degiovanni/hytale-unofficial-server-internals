---
description: Architectural reference for DelayedEntitySystem
---

# DelayedEntitySystem

**Package:** com.hypixel.hytale.component.system.tick
**Type:** Abstract System Template

## Definition
```java
// Signature
public abstract class DelayedEntitySystem<ECS_TYPE> extends EntityTickingSystem<ECS_TYPE> {
```

## Architecture & Concepts
The DelayedEntitySystem is an abstract base class designed to throttle the execution frequency of an Entity-Component-System (ECS) process. It serves as a foundational building block for systems that do not need to run on every single frame of the main game loop, such as periodic AI updates, environmental effects, or low-frequency network synchronization.

Architecturally, it acts as a temporal decorator over the standard EntityTickingSystem. Instead of executing its primary logic on every invocation of the `tick` method, it accumulates the incoming delta time (`dt`). Only when this accumulated time meets or exceeds a developer-defined `intervalSec` does it delegate the `tick` call to its parent, EntityTickingSystem, which in turn processes the relevant entities.

This mechanism is achieved by managing a private, per-system `Resource` within the ECS `Store`. This resource, an instance of the inner class `Data`, is responsible for tracking the elapsed time since the last execution. This design elegantly decouples the system's state (the time accumulator) from the system instance itself, adhering to ECS principles where the `Store` is the single source of truth for all state.

### Lifecycle & Ownership
- **Creation:** As an abstract class, DelayedEntitySystem is never instantiated directly. A concrete implementation (e.g., `PeriodicAISystem`) must extend it and is typically instantiated once by the engine's service loader or system registry during world initialization.
- **Scope:** An instance of a concrete DelayedEntitySystem persists for the entire lifetime of the ECS `Store` or world it is registered with. Its internal time-tracking `Data` resource shares this same lifecycle.
- **Destruction:** The system and its associated `Data` resource are marked for garbage collection when the parent ECS world is destroyed.

## Internal State & Concurrency
- **State:** The DelayedEntitySystem instance itself is effectively immutable after construction; its `intervalSec` is final. The mutable state, specifically the accumulated delta time, is not stored in the system object but is managed externally within the ECS `Store` as a `Data` resource. This is a critical design choice that keeps the system logic stateless.
- **Thread Safety:** This class is **not thread-safe**. The `tick` method involves a read-modify-write operation on the `Data` resource within the `Store`. The ECS framework guarantees that all systems for a given `Store` are ticked sequentially on a single, designated thread. Invoking `tick` concurrently from multiple threads on the same `Store` will lead to race conditions and unpredictable behavior.

**WARNING:** Do not attempt to parallelize system updates by calling `tick` from multiple threads. The design assumes a single-threaded, sequential update loop.

## API Surface
The public contract is minimal, primarily exposing the core `tick` method inherited and overridden from its parent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, systemIndex, store) | void | O(1) | Accumulates delta time. If the configured interval has elapsed, triggers the parent `tick` logic. |
| getIntervalSec() | float | O(1) | Returns the configured update interval in seconds. |
| getResourceType() | ResourceType | O(1) | Returns the unique identifier for the internal `Data` resource used for time tracking. |

## Integration Patterns

### Standard Usage
The primary pattern is to extend this class, provide an interval in the constructor, and implement the entity-specific logic in `tickEntity`. The framework handles the rest.

```java
// A concrete system that processes AI logic for entities every 1.5 seconds.
public class AIUpdateSystem extends DelayedEntitySystem<GameEntity> {

    // Define the update interval via the super constructor.
    public AIUpdateSystem() {
        super(1.5f);
    }

    // This logic is guaranteed to run only once every 1.5 seconds.
    // The 'dt' parameter here will be the total accumulated time (~1.5f),
    // not the frame's delta time.
    @Override
    public void tickEntity(float dt, int systemIndex, GameEntity entity, Store<GameEntity> store) {
        // Perform expensive AI pathfinding or behavior tree updates.
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Misinterpreting Delta Time:** Do not use the `dt` parameter within an overridden `tickEntity` method as if it were the frame's delta time. It represents the *total accumulated time* since the last execution, which will be approximately equal to `intervalSec`. Using it for fine-grained physics or animation will produce incorrect, jerky motion.
- **Stateful Subclasses:** Avoid storing mutable state as instance fields in a subclass. This violates the stateless nature of ECS systems and can cause issues with hot-reloading or serialization. All state should be stored in Components or Resources.
- **Manual Time Management:** Do not attempt to manually manage the internal `Data` resource. It is an implementation detail and is managed entirely by the DelayedEntitySystem's `tick` method.

## Data Pipeline
The flow of a `tick` call demonstrates how the delay mechanism is implemented. The system acts as a gatekeeper for the parent `EntityTickingSystem` logic.

> Flow:
> Game Loop Tick -> Engine dispatches to `DelayedEntitySystem.tick()` -> Retrieve `Data` resource from `Store` -> Accumulate delta time -> **Interval Check** -> (If false, return) -> (If true, reset accumulator) -> Delegate to `super.tick()` -> `EntityTickingSystem` iterates entities -> `ConcreteSystem.tickEntity()` is called for each entity.

