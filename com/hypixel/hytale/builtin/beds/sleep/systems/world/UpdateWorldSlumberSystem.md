---
description: Architectural reference for UpdateWorldSlumberSystem
---

# UpdateWorldSlumberSystem

**Package:** com.hypixel.hytale.builtin.beds.sleep.systems.world
**Type:** System Component

## Definition
```java
// Signature
public class UpdateWorldSlumberSystem extends TickingSystem<EntityStore> {
```

## Architecture & Concepts
The UpdateWorldSlumberSystem is a server-side system within Hytale's Entity Component System (ECS) framework. It is the primary driver for the time acceleration mechanic when one or more players are sleeping in a world.

This system functions as a specialized state processor. It remains dormant until the world's global `WorldSomnolence` resource enters the `WorldSlumber` state. Once active, its sole responsibility is to advance game time rapidly and monitor for conditions that would end the slumber.

Key responsibilities include:
- Incrementally advancing the slumber progress based on real-world time (delta time).
- Calculating and applying a new, fast-forwarded game time to the `WorldTimeResource` upon completion.
- Transitioning the world state from `WorldSlumber` back to `Awake`.
- Updating the `PlayerSomnolence` component for all sleeping players to a wake-up state.

This system is a critical bridge between a global world state (slumbering) and the fundamental server mechanic of timekeeping. It ensures that the consequences of sleeping are applied atomically and consistently across all players and the world itself.

### Lifecycle & Ownership
- **Creation:** Instantiated by the server's System Scheduler during world initialization. This is an engine-level process; developers do not manually create instances of this system.
- **Scope:** The lifecycle of an UpdateWorldSlumberSystem instance is tightly coupled to a single `World` instance. It persists as long as its associated world is loaded and active on the server.
- **Destruction:** The instance is marked for garbage collection when the world it belongs to is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** The UpdateWorldSlumberSystem is **stateless**. It does not maintain any internal fields that persist across ticks. All data, such as slumber progress and target time, is read directly from external ECS resources (`WorldSomnolence`, `WorldTimeResource`) and components (`PlayerSomnolence`) during each execution of the `tick` method. This stateless design is crucial for predictability and robustness within the ECS paradigm.

- **Thread Safety:** The `tick` method itself is expected to be invoked by a single, main game-loop thread. However, the system leverages `forEachEntityParallel` to process player state changes concurrently. Thread safety is guaranteed through the use of a `commandBuffer`. Instead of modifying components directly, which would create race conditions, the system enqueues state-change commands. These commands are then executed safely by the engine at a designated synchronization point after all parallel processing is complete.

## API Surface
The public API is defined by its contract with the `TickingSystem` superclass. Direct invocation is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, systemIndex, store) | void | O(P) | Executes one cycle of the slumber logic. Complexity is proportional to the number of players (P) due to the `isSomeoneAwake` check. Throws exceptions if required resources are not registered. |

## Integration Patterns

### Standard Usage
This system is not designed to be invoked directly. Its functionality is triggered by manipulating the ECS world state. To initiate the time acceleration managed by this system, another system must set the `WorldSomnolence` resource to an instance of `WorldSlumber`.

```java
// Another system (e.g., one that processes a player entering a bed)
// would trigger the UpdateWorldSlumberSystem by changing the world state.

// This code is conceptual and would exist in a different system.
WorldSomnolence worldSomnolence = store.getResource(WorldSomnolence.getResourceType());
WorldSlumber newSlumber = createNewSlumberState(...);
worldSomnolence.setState(newSlumber);

// On subsequent ticks, the engine will automatically run UpdateWorldSlumberSystem,
// which will now detect the WorldSlumber state and begin its logic.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new UpdateWorldSlumberSystem()`. The engine's System Scheduler is solely responsible for its lifecycle. Manually creating an instance will result in a non-functional object that is not registered to receive ticks.
- **Manual Invocation:** Do not call the `tick` method directly. This bypasses the engine's scheduler, synchronization points, and command buffer processing, which can lead to severe state corruption, race conditions, and server instability.
- **Stateful Logic:** Do not modify this system to store state in member variables. All state should be managed through ECS components and resources to maintain architectural integrity.

## Data Pipeline
The system's operation is a clear sequence of reading from the world state, processing, and writing deferred changes back to the world state.

> Flow:
> **Input Trigger:** `WorldSomnolence` resource is in `WorldSlumber` state.
> 1. The System Scheduler invokes the `tick` method.
> 2. The system reads the current `WorldSlumber` state from the `WorldSomnolence` resource.
> 3. It checks for an exit condition by iterating over all `PlayerSomnolence` components via `isSomeoneAwake`.
> 4. If the slumber continues, it calculates the new progress and exits.
> 5. If the slumber ends, it calculates the final wake-up time.
> 6. It writes a state change command for the `WorldSomnolence` resource (to `Awake`).
> 7. It writes a state change command for the `WorldTimeResource` (setting the new game time).
> 8. It enqueues component update commands to the `commandBuffer` for each sleeping player.
> **Output:** The `commandBuffer` is populated with commands to update world resources and player components, which are applied atomically by the engine.

