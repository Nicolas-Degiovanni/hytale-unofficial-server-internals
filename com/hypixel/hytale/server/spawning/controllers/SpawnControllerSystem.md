---
description: Architectural reference for SpawnControllerSystem
---

# SpawnControllerSystem

**Package:** com.hypixel.hytale.server.spawning.controllers
**Type:** System Framework

## Definition
```java
// Signature
public abstract class SpawnControllerSystem<J extends SpawnJob, T extends SpawnController<J>> extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The SpawnControllerSystem is an abstract base class that serves as the execution engine for server-side NPC spawning logic. It is a foundational component within Hytale's Entity-Component-System (ECS) architecture, inheriting from EntityTickingSystem to ensure its logic is integrated directly into the main server game loop.

This class embodies the principle of separating state from behavior. The *state* and *rules* for spawning (e.g., mob caps, spawn conditions, biomes) are held within a corresponding SpawnController instance. This system acts as the *behavioral driver*; on each server tick, it queries the state of the world and its associated SpawnController to determine if new spawn operations should be initiated.

Its core responsibility is to bridge the game loop with the spawning rules, periodically triggering the creation of SpawnJob instances which are then processed by other downstream systems. Concrete implementations, such as a hypothetical HostileSpawnControllerSystem, will target a specific type of SpawnController to manage a distinct category of NPC spawning.

### Lifecycle & Ownership
-   **Creation:** Concrete implementations of this class are discovered and instantiated by the server's central SystemRegistry during world initialization. It is a framework-managed object, not intended for manual creation.
-   **Scope:** An instance of a SpawnControllerSystem is scoped to a running World. Its lifecycle is directly coupled to the server world it operates within, persisting for the entire session.
-   **Destruction:** The system is marked for garbage collection when its associated World is unloaded or during a full server shutdown.

## Internal State & Concurrency
-   **State:** The SpawnControllerSystem is designed to be **stateless**. It does not maintain any internal state or cache data across ticks. All decisions are made using the real-time data provided by the World and the SpawnController passed into its methods during a tick.
-   **Thread Safety:** This class is **not thread-safe** and must only be accessed from the main server thread. As an EntityTickingSystem, the engine guarantees that its methods are invoked sequentially within the server's primary game loop. Any attempt to invoke its methods from an asynchronous task or external thread will result in world state corruption and severe concurrency violations.

## API Surface
The public contract is defined by its role as a ticking system. The primary interaction points are the protected methods intended for extension by subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tickController(spawnController, store) | protected void | O(N) | The core logic hook. Evaluates spawning conditions and triggers job creation. N is the max active jobs. |
| prepareSpawnJobGeneration(controller, accessor) | protected abstract void | - | Abstract template method. Subclasses must implement this to perform any pre-spawn setup for a given tick. |
| createRandomSpawnJobs(controller, accessor) | protected void | O(N) | Drives a loop that requests new SpawnJob instances from the controller until the active job limit is reached. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly. Instead, they create a concrete subclass to define the spawning behavior for a new category of NPCs. The engine's SystemRegistry will automatically discover and manage the system.

```java
// A concrete implementation defines the link between a controller and the system.
// The engine will then automatically tick this system.
public class AmbientCreatureSpawnSystem extends SpawnControllerSystem<AmbientSpawnJob, AmbientSpawnController> {

    @Override
    protected void prepareSpawnJobGeneration(AmbientSpawnController controller, ComponentAccessor<EntityStore> accessor) {
        // Custom logic to prepare for spawning ambient creatures,
        // e.g., checking time of day or weather from the world state.
    }

    // The base class methods tickController and createRandomSpawnJobs are inherited
    // and provide the core spawning loop.
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance of a system using `new`. The SystemRegistry is solely responsible for the lifecycle of all systems. Manual creation will result in a non-functional system that is not registered with the game loop.
-   **Stateful Implementations:** Avoid adding mutable instance fields to concrete subclasses. Systems in an ECS architecture should be stateless, with all relevant data stored in components or passed as parameters.
-   **Asynchronous Invocation:** Never call `tickController` or other methods from a separate thread. All interactions with the world via this system must be synchronized with the main server tick.

## Data Pipeline
This system acts as an originator in the data pipeline, converting game state into discrete work units (SpawnJob). It does not typically process inbound data from other systems.

> Flow:
> Server Game Loop Tick -> **SpawnControllerSystem.tickController** -> World State Evaluation -> `SpawnController.createRandomSpawnJob` -> New SpawnJob -> Spawning Job Queue -> Job Processor System

