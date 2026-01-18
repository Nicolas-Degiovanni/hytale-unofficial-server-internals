---
description: Architectural reference for WorldSpawningSystem
---

# WorldSpawningSystem

**Package:** com.hypixel.hytale.server.spawning.world.system
**Type:** System Service

## Definition
```java
// Signature
public class WorldSpawningSystem extends TickingSystem<ChunkStore> {
```

## Architecture & Concepts
The WorldSpawningSystem is a core server-side system responsible for orchestrating the ambient spawning of Non-Player Characters (NPCs) across the game world. It functions as a high-level "director" that ensures the world's population adheres to configured limits and feels naturally distributed.

Architecturally, this system is a classic example of a decoupled, data-driven design within an Entity Component System (ECS) framework. Its primary role is not to spawn entities directly, but to make decisions and delegate the work. It operates by creating **SpawnJobData** components and attaching them to chunk entities. A separate, lower-level system is then responsible for observing these job components and executing the complex logic of placing the NPC in the world.

This system's decision-making process is sophisticated, balancing multiple inputs:
-   **Global Population Caps:** It respects the server-wide `maxEnvironmentalNPCSpawns` setting.
-   **Player Presence:** Spawning is completely halted if no players are in the world.
-   **Expected vs. Actual Counts:** It constantly compares the desired number of NPCs (the "expected" count, derived from world state and environment data) against the current population (the "actual" count).
-   **Environment & Biome Weighting:** It uses a weighted random selection to choose which environment (e.g., forest, desert) should receive the next spawn, preventing over-population in one area.
-   **Chunk-level State:** It considers the status of individual chunks, including spawn cooldowns and whether a chunk is considered "fully populated".

By creating abstract spawn jobs instead of entities, the system isolates the high-level population logic from the low-level placement and instantiation logic, improving maintainability and performance.

### Lifecycle & Ownership
-   **Creation:** A single instance of WorldSpawningSystem is created and managed by the server's ECS framework for each active game world. It is typically instantiated during the world's initialization sequence, receiving its dependencies (ComponentType and ResourceType handles) from the central ECS registry.
-   **Scope:** The system's lifecycle is tightly bound to its parent **World** instance. It persists as long as the world is loaded and active.
-   **Destruction:** The instance is marked for garbage collection when the corresponding world is unloaded. The ECS framework manages its removal from the list of active ticking systems.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. Its fields are final references to ECS type definitions, which are immutable after construction. All mutable state it reads and modifies (e.g., NPC counts, active jobs, chunk cooldowns) is stored externally in ECS components like **ChunkSpawnData** and resources like **WorldSpawnData**. This design is fundamental to ECS, keeping systems as pure logic processors.
-   **Thread Safety:** This system is **not thread-safe** and must only be executed on the main thread for its assigned world. The `TickingSystem` contract guarantees that the `tick` method is called sequentially from a single, synchronized game loop. The use of `ThreadLocalRandom` further indicates an expectation of single-threaded access within a tick cycle. Any external modification of the component data it accesses during a tick will lead to concurrency violations and unpredictable behavior.

## API Surface
The public contract is exclusively for the ECS framework. Developers should never call these methods directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, systemIndex, store) | void | Bounded Iterative | The main entry point called by the game loop once per tick. Evaluates world spawn state and creates new **SpawnJobData** components if conditions are met. Complexity is proportional to the number of active environments but capped by `maxActiveJobs`. |

## Integration Patterns

### Standard Usage
Direct interaction with this system is not a standard development pattern. The ECS framework automatically registers and executes it as part of the server's main game loop for each world. Its operation is implicit and configuration-driven. Developers influence its behavior by modifying gameplay configuration assets and NPC spawn definitions, not by calling its methods.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new WorldSpawningSystem()`. The system requires dependencies from the ECS registry that can only be provided during framework-managed initialization.
-   **Manual Ticking:** Never call the `tick` method manually. Doing so would bypass the world's update cycle, break thread safety guarantees, and corrupt game state.
-   **External State Mutation:** Modifying `WorldSpawnData` or `ChunkSpawnData` from another thread while this system is running will cause critical race conditions, leading to incorrect NPC counts and potential server crashes.

## Data Pipeline
The system acts as a stateful filter and job creator. It does not transform a stream of data but rather evaluates the holistic state of the world to generate discrete work orders.

> Flow:
> Game Loop Tick → **WorldSpawningSystem.tick()** → Read Global State (WorldSpawnData, Player Count) → Check Population Caps → Select Weighted Environment → Select Weighted Chunk → **Create SpawnJobData Component** → Attach Component to Chunk Entity → Downstream Job Executor System

