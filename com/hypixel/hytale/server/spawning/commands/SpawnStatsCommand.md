---
description: Architectural reference for SpawnStatsCommand
---

# SpawnStatsCommand

**Package:** com.hypixel.hytale.server.spawning.commands
**Type:** Transient

## Definition
```java
// Signature
public class SpawnStatsCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The **SpawnStatsCommand** is a server-side administrative command designed for diagnostics and monitoring of the world's NPC spawning systems. It does not participate in the core game simulation loop; instead, it functions as a high-level, read-only reporting tool that provides a point-in-time snapshot of spawning health.

This command acts as a data aggregator, pulling information from two primary spawning mechanisms:
1.  **Dynamic Environment Spawning:** The procedural system that spawns NPCs based on environmental factors like biome, time, and light level.
2.  **Static Marker Spawning:** The deterministic system that spawns NPCs at predefined locations using **SpawnMarkerEntity** instances.

By querying authoritative data sources like **WorldSpawnData** and the global **EntityStore**, the command generates detailed reports directly to the server log. This decouples the complex, performance-sensitive task of tracking statistics from the act of reporting them, allowing administrators to inspect the system's state on-demand without impacting performance.

## Lifecycle & Ownership
- **Creation:** A prototype instance of **SpawnStatsCommand** is created and registered by the server's **CommandSystem** during the server bootstrap sequence. The command itself is effectively a singleton from the perspective of the registration system.
- **Scope:** The command object's operational lifecycle is confined to a single execution. When an administrator invokes the command, the system uses the registered instance to call the **execute** method. All internal state required for the report is created, used, and discarded within this single method call.
- **Destruction:** As a transient-use object with no persistent state between executions, the command instance itself persists for the life of the server, but its operational context is destroyed immediately after the **execute** method returns. Any local variables or data collections are then eligible for garbage collection.

## Internal State & Concurrency
- **State:** **SpawnStatsCommand** is fundamentally stateless between invocations. Its fields, such as **environmentsArg** and **markersArg**, are final configurations set in the constructor and are immutable. During execution, it allocates temporary data structures (e.g., **AtomicInteger**, **Object2IntOpenHashMap**) to aggregate statistics. This state is strictly local to the **execute** method's stack frame and does not persist.

- **Thread Safety:** This command is designed to be executed safely. The **execute** method is called from a dedicated command processing thread. It initiates several parallel read operations on the world's entity data using methods like **forEachEntityParallel** and **forEachChunk**. The underlying data stores are architected for safe concurrent reads. Aggregation is managed using thread-safe collectors like **AtomicInteger** to prevent race conditions during data collection from multiple worker threads.

    **Warning:** While the command itself is thread-safe, running it on a world with extremely high entity counts may introduce significant CPU load due to the full-scan nature of its queries.

## API Surface
The primary contract is the inherited **execute** method. The command's behavior is configured through command-line flags, not a programmatic API.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(E + C) | Gathers and logs spawning statistics. Complexity is linear, proportional to the number of total Entities (E) and loaded Chunks (C) in the world, as it performs full data scans. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use by other game systems. It is designed to be invoked by a server administrator through the server console or an in-game chat command.

```
// Conceptual invocation via server console
/spawning stats --environments --verbose
/spawning stats --markers
```

The underlying system handles the invocation based on the registered command name.

```java
// The CommandSystem is responsible for resolving and executing the command.
// This code is conceptual and represents the system's internal logic.

CommandSystem commandSystem = server.getCommandSystem();
CommandContext context = create_context_from_input("/spawning stats --environments");

// The system finds the registered SpawnStatsCommand and invokes it.
commandSystem.execute(context);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use **new SpawnStatsCommand()**. The command's lifecycle is exclusively managed by the server's **CommandSystem**. Manual instantiation bypasses registration and context injection, rendering the object non-functional.
- **Game Logic Integration:** Never call this command from core game logic, such as an AI behavior or a block update. It is a high-cost, blocking diagnostic tool, and its execution can cause noticeable hitches in the game loop.
- **Stateful Modification:** Do not modify the class to store state between executions. This violates its design as a transient command and will lead to incorrect data and concurrency bugs in a live server environment.

## Data Pipeline
**SpawnStatsCommand** is a terminal endpoint in a data flow; it aggregates and reports but does not pass data to other systems. The flow represents data being pulled *into* the command for processing.

**Environment Stats Flow:**
> Flow:
> WorldSpawnData & ChunkSpawnData -> **SpawnStatsCommand** -> Parallel Aggregation -> HytaleLogger -> Server Log

**Spawn Marker Stats Flow:**
> Flow:
> EntityStore (NPCEntity, SpawnMarkerEntity) -> **SpawnStatsCommand** -> Parallel Aggregation -> HytaleLogger -> Server Log

