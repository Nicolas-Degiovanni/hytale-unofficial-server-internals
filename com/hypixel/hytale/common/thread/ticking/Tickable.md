---
description: Architectural reference for the Tickable interface, the core contract for game loop updates.
---

# Tickable

**Package:** com.hypixel.hytale.common.thread.ticking
**Type:** Contract Interface

## Definition
```java
// Signature
public interface Tickable {
   void tick(float var1);
}
```

## Architecture & Concepts
The Tickable interface is one of the most fundamental contracts within the Hytale engine architecture. It embodies the "Update Method" pattern, providing a standardized entry point for the main game loop to update the state of disparate game systems over time.

This interface acts as a decoupling mechanism. The core scheduler, or Ticker, does not need to know the concrete type of an object it is updating. It only needs to know that the object conforms to the Tickable contract. This allows any system—from entity AI and physics simulation to animation controllers and particle systems—to register itself for periodic updates without creating tight coupling with the engine's central timing mechanism.

Objects implementing Tickable are expected to perform all their per-frame logic within the `tick` method. The float parameter, representing delta time, is critical for ensuring that all time-based logic (e.g., movement, cooldowns) is independent of the client's or server's frame rate.

## Lifecycle & Ownership
As an interface, Tickable itself has no lifecycle. The lifecycle of its *implementations* is paramount to engine stability.

- **Creation:** An object implementing Tickable is created by its owning system (e.g., an EntityManager creates an Entity). Upon creation, it is typically registered with a central Ticker or a world-specific scheduler.
- **Scope:** The lifetime of a Tickable implementation is bound to its owner. For example, a Tickable component on an entity persists only as long as that entity exists in the world.
- **Destruction:** Upon destruction of the owning object, the Tickable implementation **must** be deregistered from its scheduler. Failure to do so is a critical error that will lead to memory leaks and attempts to update stale or null object references, inevitably causing engine crashes.

## Internal State & Concurrency
- **State:** Implementations of Tickable are, by their very nature, mutable. The primary purpose of the `tick` method is to read the current state of the object and the game world, and then mutate the object's state for the next frame.
- **Thread Safety:** **CRITICAL WARNING:** Implementations of Tickable are assumed to be **thread-unsafe**. The `tick` method is almost exclusively invoked by a single, dedicated game thread (the "main thread" or "tick thread"). All state modification must occur on this thread. Any interaction with other threads (e.g., networking, file I/O) must be managed through thread-safe queues or other synchronization primitives to prevent catastrophic race conditions and data corruption. Direct state access from other threads is strictly forbidden.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(float deltaTime) | void | O(N) | Executes a single unit of work for the current frame. The complexity is entirely dependent on the implementation. Throws NullPointerException if the underlying object is destroyed but not deregistered. |

## Integration Patterns

### Standard Usage
The standard pattern involves implementing the interface on a component or system and registering it with the appropriate scheduler during initialization.

```java
// Example of a simple component implementing Tickable
public class PlayerMovementComponent implements Tickable {
    private Player owner;
    private Vector3f velocity;

    public PlayerMovementComponent(Player owner) {
        this.owner = owner;
        // Initialization...
    }

    @Override
    public void tick(float deltaTime) {
        // Update position based on velocity and delta time
        Vector3f position = owner.getPosition();
        position.add(velocity.mul(deltaTime));
        owner.setPosition(position);
    }
}

// Registration (elsewhere in the engine)
Player player = world.spawnPlayer();
PlayerMovementComponent movement = new PlayerMovementComponent(player);
world.getTicker().register(movement);
```

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Never call the `tick` method directly. Doing so circumvents the engine's scheduler, leading to incorrect delta time values, unpredictable update order, and severe timing bugs. Always let the Ticker manage the update cycle.
- **Blocking Operations:** The `tick` method must execute quickly. Any long-running or blocking operations (e.g., disk access, complex pathfinding, network requests) will freeze the entire game thread, resulting in catastrophic performance degradation. Offload heavy work to background threads.
- **Ignoring Delta Time:** Failing to use the `deltaTime` parameter for time-based calculations will couple your logic to the frame rate, causing it to run faster or slower on different machines.

## Data Pipeline
Tickable is part of a control flow, not a data pipeline. It is the mechanism by which the engine propagates time throughout its systems.

> Flow:
> Main Game Loop -> Scheduler/Ticker -> **Tickable.tick(deltaTime)** -> Game State Mutation -> Subsequent Systems (e.g., Rendering)

