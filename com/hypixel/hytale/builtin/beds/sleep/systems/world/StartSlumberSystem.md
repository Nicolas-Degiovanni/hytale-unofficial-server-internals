---
description: Architectural reference for StartSlumberSystem
---

# StartSlumberSystem

**Package:** com.hypixel.hytale.builtin.beds.sleep.systems.world
**Type:** Engine System

## Definition
```java
// Signature
public class StartSlumberSystem extends DelayedSystem<EntityStore> {
```

## Architecture & Concepts
The **StartSlumberSystem** is a core state machine orchestrator within the server-side Entity Component System (ECS) framework, responsible for managing the world's transition into a collective sleep state. It operates on a consensus model: the system periodically polls the sleep-readiness of all connected players and, upon unanimous agreement, triggers a global time acceleration event.

This system acts as the central authority for initiating the "slumber" phase of the day-night cycle. It decouples the individual player action of using a bed (which sets their state to **NoddingOff**) from the world-level event of fast-forwarding time. By centralizing this logic, it ensures that the time transition is synchronized for all players and only occurs when all preconditions are met.

It is a *delayed* system, meaning it does not execute on every game tick. Instead, it runs on a fixed, low-frequency interval (0.3 seconds) to reduce performance overhead, as the conditions for sleeping do not need to be checked with high frequency.

### Lifecycle & Ownership
-   **Creation:** An instance of **StartSlumberSystem** is instantiated automatically by the server's ECS System Manager when a world is initialized. It is registered as part of the standard gameplay systems.
-   **Scope:** The system's lifecycle is tightly bound to its parent **World**. It persists for the entire duration that the world is loaded and active on the server.
-   **Destruction:** The instance is marked for garbage collection and destroyed when the associated **World** is unloaded, for example, during a server shutdown or if the world becomes inactive.

## Internal State & Concurrency
-   **State:** The **StartSlumberSystem** is fundamentally stateless. It does not maintain any internal member variables that persist across ticks. All stateful information, such as player sleep status or world time, is read directly from ECS **Components** (e.g., **PlayerSomnolence**) and **Resources** (e.g., **WorldSomnolence**, **WorldTimeResource**) within the provided **EntityStore**.

-   **Thread Safety:** This system is designed for concurrent execution. The primary entry point, **delayedTick**, is invoked by a single engine thread. However, when updating player components, it utilizes the **forEachEntityParallel** method. This method processes entities across multiple worker threads.

    **WARNING:** To prevent race conditions, all component modifications within the parallel loop are queued into a thread-safe **commandBuffer**. These commands are then applied synchronously by the engine after the parallel processing phase is complete. The static method **isReadyToSleep** is inherently thread-safe as it is a pure function that only reads data from the **ComponentAccessor**.

## API Surface
The primary public contract is a static utility method for querying player state. The **delayedTick** method is an engine-level override and should not be invoked directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isReadyToSleep(store, ref) | static boolean | O(1) | Checks if a specific player entity is in a state considered ready for world slumber. This is the main integration point for other systems. |

## Integration Patterns

### Standard Usage
This system runs automatically and requires no direct invocation. Other game systems that need to know if a player is ready to sleep should use the static utility method.

```java
// Example from another system checking a player's status
Ref<EntityStore> playerRef = ...; // Get player reference
ComponentAccessor<EntityStore> store = ...; // Get component accessor

boolean canSleep = StartSlumberSystem.isReadyToSleep(store, playerRef);

if (canSleep) {
    // Logic for a player who is ready to sleep
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use **new StartSlumberSystem()**. The engine's System Manager is solely responsible for the lifecycle of this class. Manual instantiation will result in a non-functional object that is not registered with the game loop.
-   **Manual Invocation:** Never call the **delayedTick** method directly. This bypasses the engine's scheduler and thread management, which will lead to unpredictable behavior, state corruption, and severe concurrency issues.

## Data Pipeline
The **StartSlumberSystem** functions as a processor that reads distributed player states and writes a new global world state.

> Flow:
> Engine Scheduler -> **StartSlumberSystem.delayedTick** -> Reads **PlayerSomnolence** Component for each player -> Reads **WorldSomnolence** Resource -> If all players are ready -> **StartSlumberSystem** calculates wake-up time -> Writes new **WorldSlumber** state to **WorldSomnolence** Resource -> Writes new **PlayerSleep.Slumber** state to each player's **PlayerSomnolence** Component via Command Buffer.

