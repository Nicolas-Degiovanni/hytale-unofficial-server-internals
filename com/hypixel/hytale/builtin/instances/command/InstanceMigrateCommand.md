---
description: Architectural reference for InstanceMigrateCommand
---

# InstanceMigrateCommand

**Package:** com.hypixel.hytale.builtin.instances.command
**Type:** Transient

## Definition
```java
// Signature
public class InstanceMigrateCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The InstanceMigrateCommand is an administrative utility responsible for performing offline data schema migrations on "instance" worlds. Instances are pre-defined world templates that can be instantiated for various gameplay purposes. This command is not part of the real-time game loop; it is a maintenance tool executed by a server operator to ensure that world data stored on disk is compatible with the current version of the server software.

Its core architectural function is to act as an orchestrator for the data migration pipeline. It operates through the following high-level process:
1.  **Isolate:** For each discoverable instance asset, it creates a temporary, non-ticking, and non-interactive world within the server's Universe. This sandboxes the migration process, preventing interference with live game worlds and disabling game logic like physics or AI.
2.  **Iterate:** It queries the world's storage layer to get a complete index of all chunks present on disk.
3.  **Delegate:** For each chunk, it loads the raw data and passes it to a series of registered migration systems (e.g., ChunkColumnMigrationSystem, EntityModule.MigrationSystem). These specialized systems contain the actual logic for upgrading specific data components from an old format to a new one.
4.  **Persist:** If any migration system modifies a chunk, the command instructs the storage layer to write the updated chunk data back to disk, overwriting the old version.
5.  **Cleanup:** Once all chunks in an instance are processed, the temporary world is removed from the Universe.

The entire operation is heavily asynchronous and parallelized at the chunk level, leveraging CompletableFutures to maximize I/O and CPU throughput during the migration.

## Lifecycle & Ownership
-   **Creation:** The command is registered with the server's command system on startup. A new instance of this class is created by the command dispatcher each time an administrator executes the corresponding command.
-   **Scope:** The object's lifetime is scoped to a single command execution. It is created, its `executeAsync` method is called, and it becomes eligible for garbage collection once the returned CompletableFuture completes.
-   **Destruction:** The object itself is short-lived and managed by the garbage collector. However, it is responsible for the explicit destruction of the temporary worlds it creates for migration via `Universe.removeWorld`. Failure to complete this step would result in a memory leak for the duration of the server's uptime.

## Internal State & Concurrency
-   **State:** The InstanceMigrateCommand class is stateless. All necessary state, such as the command context for sending messages and counters for tracking progress, is created within the scope of the `executeAsync` method. The command's primary function is to mutate the persistent state of world files on the filesystem.
-   **Thread Safety:** This class orchestrates a highly concurrent operation but is not itself designed to be thread-safe. The command framework guarantees that a single instance is only used for one execution at a time. The core logic within `migrateInstance` dispatches thousands of parallel tasks for chunk loading, processing, and saving. Thread safety for progress reporting across these parallel tasks is correctly managed using `AtomicLong` counters. The underlying storage systems (`IChunkLoader`, `IChunkSaver`) are expected to be thread-safe to support this parallel execution model.

**WARNING:** Executing this command can place an extremely high I/O load on the disk subsystem. It is intended for offline maintenance periods.

## API Surface
The public API is defined by its parent, `AbstractAsyncCommand`, and is invoked exclusively by the server's command handling system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context) | CompletableFuture<Void> | O(N * M) | Orchestrates the entire migration for all known instances. N is the total number of chunks across all instances, and M is the number of registered migration systems. Returns a future that completes when all operations are finished or an error occurs. |

## Integration Patterns

### Standard Usage
This command is not intended for programmatic use by other systems. It is designed to be executed by a server administrator through the server console.

```
// Console Command
instances migrate
```

The command provides progress feedback directly to the console or in-game chat of the executing user.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new InstanceMigrateCommand()`. The command system handles instantiation and injection of the `CommandContext`. Manual creation will result in a non-functional object.
-   **Manual Invocation:** Do not call the `executeAsync` method directly. This bypasses the command framework's permission checks, context setup, and error handling.
-   **Execution on a Live Server:** While the command attempts to isolate its workload, the intense disk I/O can severely degrade the performance of the entire server. This command should only be run during designated maintenance windows.

## Data Pipeline
The flow of data through this command is a destructive, read-modify-write cycle targeting the filesystem.

> Flow:
> 1. Administrator executes command.
> 2. `InstancesPlugin` -> Provides list of instance asset paths.
> 3. `WorldConfig.load` -> Reads `instance.bson` from **Disk**.
> 4. `Universe.makeWorld` -> Creates temporary in-memory World object.
> 5. `IChunkLoader.getIndexes` -> Reads chunk manifest from **Disk**.
> 6. `IChunkLoader.loadHolder` -> Reads raw chunk data from **Disk** into an in-memory `Holder`.
> 7. **InstanceMigrateCommand** -> Passes `Holder` to `MigrationSystem`s.
> 8. `MigrationSystem` -> Modifies component data within the in-memory `Holder`.
> 9. `IChunkSaver.saveHolder` -> Writes modified `Holder` data back to **Disk**.
> 10. `Universe.removeWorld` -> Deallocates the in-memory World object.

