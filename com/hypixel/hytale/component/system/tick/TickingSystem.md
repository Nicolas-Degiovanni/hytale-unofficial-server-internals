---
description: Architectural reference for TickingSystem
---

# TickingSystem

**Package:** com.hypixel.hytale.component.system.tick
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class TickingSystem<ECS_TYPE> extends System<ECS_TYPE> implements TickableSystem<ECS_TYPE> {
```

## Architecture & Concepts
The TickingSystem is a foundational abstract class within the Hytale Entity-Component-System (ECS) architecture. It serves as the primary template for any game logic system that must execute code on a recurring, time-sensitive basis, synchronized with the main game loop.

By extending the base System and implementing the TickableSystem interface, this class establishes a strict contract for all time-dependent logic. It acts as the bridge between the engine's central scheduler, which manages the game loop, and the concrete systems that implement game features like physics, animation, AI, or environmental effects.

Subclasses of TickingSystem are responsible for querying and manipulating a specific set of components, identified by the generic ECS_TYPE, via the provided component Store. The core responsibility of a TickingSystem is to contain the logic that advances the state of its associated components from one frame to the next.

### Lifecycle & Ownership
- **Creation:** A TickingSystem is an abstract class and is never instantiated directly. Concrete implementations (e.g., PhysicsSystem, AnimationSystem) are discovered and instantiated by the engine's SystemRegistry during the world-loading or session-initialization phase.
- **Scope:** An instance of a concrete TickingSystem persists for the entire lifetime of the game world or session it is registered with. Its lifecycle is directly coupled to its parent world.
- **Destruction:** Instances are marked for garbage collection when the associated game world is unloaded or the server/client shuts down. The SystemRegistry manages this process.

## Internal State & Concurrency
- **State:** This base class is inherently stateless. However, concrete subclasses are expected to be highly stateful, as they are responsible for managing the state of game components. They may also cache queries or maintain internal data structures for performance optimization.
- **Thread Safety:** **Not thread-safe.** The tick method is designed to be invoked exclusively by the main game loop thread. Invoking tick from any other thread will lead to race conditions, data corruption, and engine instability. All state within a TickingSystem subclass must be considered thread-local to the game loop.

**WARNING:** Any asynchronous operations required by a system must be carefully managed. Offload work to other threads, but ensure all state mutations on the component Store occur only within the tick method's execution context.

## API Surface
The public contract is minimal, consisting of the single abstract method that all subclasses must implement.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(deltaTime, tickCount, componentStore) | void | O(N) | Executes one cycle of the system's logic. Complexity is typically linear, proportional to the number of entities the system must process. This method is the primary entry point and is called once per game loop iteration. |

## Integration Patterns

### Standard Usage
Developers do not interact with TickingSystem directly. Instead, they create concrete game systems by extending it and implementing the abstract tick method.

```java
// Example: A system to apply gravity to all entities with a PhysicsComponent.
public class GravitySystem extends TickingSystem<PhysicsComponent> {

    private static final float GRAVITY_ACCELERATION = -9.81f;

    @Override
    public void tick(float deltaTime, int tickCount, @Nonnull Store<PhysicsComponent> store) {
        // The 'store' provides safe access to all PhysicsComponent instances.
        for (PhysicsComponent physics : store.getAll()) {
            if (physics.isAffectedByGravity()) {
                float newVelocity = physics.getVerticalVelocity() + (GRAVITY_ACCELERATION * deltaTime);
                physics.setVerticalVelocity(newVelocity);
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new GravitySystem()`. Systems must be registered with and managed by the engine's SystemRegistry to ensure proper lifecycle and injection into the game loop.
- **Blocking Operations:** The tick method must execute quickly and be non-blocking. Performing file I/O, complex pathfinding, or synchronous network calls within tick will freeze the entire game thread, causing catastrophic performance degradation.
- **Storing External State:** Avoid storing direct references to entities or components from other systems. Always re-query the component Store within the tick method to get the most current state and avoid memory leaks or stale data.

## Data Pipeline
The TickingSystem is a critical processing stage in the main game loop's data flow. It consumes time-delta information and the current component state, and its output is the updated component state for the next frame.

> Flow:
> Game Loop Scheduler -> **TickingSystem.tick(deltaTime, ...)** -> Read from Component Store -> Process Logic -> Write to Component Store -> Render/Network Stage

