---
description: Architectural reference for SpawnController
---

# SpawnController

**Package:** com.hypixel.hytale.server.spawning.controllers
**Type:** Transient

## Definition
```java
// Signature
public abstract class SpawnController<T extends SpawnJob> {
```

## Architecture & Concepts
The SpawnController is an abstract base class that serves as the brain for all entity spawning logic within a specific World instance. It is not a global system; rather, a distinct instance of a SpawnController subclass is responsible for managing the lifecycle of NPCs and other spawned entities for a single World.

Its primary architectural role is to act as a stateful manager and factory. It continuously compares the desired number of NPCs, *expectedNPCs*, against the current population, *actualNPCs*, to determine if new spawning operations are required.

A key feature of this class is its implementation of an object pool for SpawnJob instances. It maintains two collections: *activeJobs* for currently executing spawn tasks and *idleJobs* for completed tasks that can be recycled. This pooling strategy is a critical performance optimization, minimizing garbage collection pressure by reusing job objects instead of constantly allocating new ones.

The abstract nature of the class, particularly the `createRandomSpawnJob` method, enforces a template method pattern. The SpawnController provides the generic control loop and state management, while concrete subclasses (e.g., OverworldSpawnController, DungeonSpawnController) provide the specific logic for *what* to spawn and *where* to spawn it.

## Lifecycle & Ownership
- **Creation:** A SpawnController instance is created by the server's central spawning system, likely the SpawningPlugin, when a World is first loaded and initialized. It is tightly coupled to this parent World.
- **Scope:** The controller's lifetime is strictly bound to its associated World. It persists in memory as long as the World is active and loaded on the server.
- **Destruction:** The object is eligible for garbage collection when its associated World is unloaded. There is no explicit `destroy` or `cleanup` method; its lifecycle is managed entirely by the World's lifecycle.

## Internal State & Concurrency
- **State:** This class is highly mutable and stateful. It maintains dynamic counts of NPCs and manages the collections of active and idle SpawnJob objects. The `unspawnable` and `debugSpawnFrozen` flags further represent mutable control states. The `idleJobs` queue acts as a cache for reusable SpawnJob objects.

- **Thread Safety:** **WARNING:** This class is **not thread-safe**. The internal collections, `ObjectArrayList` and `ArrayDeque`, are unsynchronized. All method calls that modify internal state, including adding jobs or creating new ones, must be performed from the same thread that owns the associated World, which is typically the main server tick thread for that world. Unsynchronized access from other threads will lead to `ConcurrentModificationException` or other catastrophic race conditions.

## API Surface
The public API is designed for state inspection and job management by a higher-level system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addIdleJob(T job) | void | O(1) | Returns a completed or unused SpawnJob to the internal object pool for recycling. |
| createRandomSpawnJob(accessor) | T | Varies | **Abstract.** Factory method that must be implemented by subclasses to create a new, specific spawn job. Returns null if no suitable job can be created. |
| getActiveJobs() | List<T> | O(1) | Returns a direct reference to the list of active jobs. **WARNING:** Modifying this list externally will corrupt the controller's state. |

## Integration Patterns

### Standard Usage
The SpawnController is designed to be driven by the main server game loop. A managing system, such as the SpawningPlugin, retrieves the controller for a given World and executes its logic as part of the world's tick cycle.

```java
// Executed within the main server tick for a specific World
SpawnController controller = world.getSpawningManager().getController();

// Check if new NPCs are needed
if (controller.getActualNPCs() < controller.getExpectedNPCs()) {
    // Attempt to create a new job
    SpawnJob newJob = controller.createRandomSpawnJob(entityStoreAccessor);

    if (newJob != null) {
        // The managing system would now process this job
        world.getSpawningManager().submitJob(newJob);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new ConcreteSpawnController(world)`. The server's spawning framework is responsible for creating and associating controllers with their respective worlds to ensure proper registration and lifecycle management.
- **Cross-Thread Modification:** Never call `addIdleJob` or access the job lists from an asynchronous task or different thread. All interactions must be synchronized with the World's main thread.
- **External List Manipulation:** Do not add or remove elements from the list returned by `getActiveJobs`. This will break the controller's internal accounting and lead to unpredictable behavior.

## Data Pipeline
The SpawnController acts as a source or initiator within the entity spawning pipeline. It does not process incoming data but rather generates new tasks based on its internal state.

> Flow:
> World Tick Event -> **SpawnController** (Evaluates State) -> `createRandomSpawnJob` (Creates Task) -> SpawnJob (Executes Logic) -> EntityStore (Persists New Entity) -> World State Update

