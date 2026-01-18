---
description: Architectural reference for MigrationModule
---

# MigrationModule

**Package:** com.hypixel.hytale.server.core.modules.migrations
**Type:** Singleton

## Definition
```java
// Signature
public class MigrationModule extends JavaPlugin {
```

## Architecture & Concepts
The MigrationModule is a core server plugin responsible for performing offline data transformations on world data. It functions as a dedicated, one-shot tool rather than a continuously running service. Its primary purpose is to upgrade world chunk data formats or apply large-scale changes that are infeasible to perform on a live server.

The module is activated via command-line arguments during server startup. Upon activation, it systematically loads every chunk from specified worlds, applies a series of registered migration routines, saves the modified chunks back to disk, and then cleanly shuts down the server.

Its design is extensible, featuring a registration pattern where other plugins or internal systems can provide their own `Migration` logic. This is achieved by mapping a string identifier to a migration constructor function, allowing for a flexible and decoupled approach to defining new data upgrade paths. The module directly integrates with the world storage layer, pausing the standard chunk saving systems to ensure data integrity during its operation.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's plugin loader during the initial bootstrap sequence. A static singleton reference, `instance`, is set within the constructor for global access.
- **Scope:** The object persists for the entire server session. However, its primary function, `runMigrations`, is designed to be a terminal operation; once the migration is complete, it initiates a server shutdown.
- **Destruction:** The object is destroyed along with all other server components when the JVM process terminates, a state it actively triggers.

## Internal State & Concurrency
- **State:** The module maintains a mutable map, `migrationCtors`, which stores the registered migration factories. This collection is populated during the server's setup phase. It also holds references to `SystemType` objects, which are handles for ECS-style systems related to chunk data.
- **Thread Safety:** This class is **not** thread-safe for configuration but is designed for concurrent execution.
    - The `register` method is not synchronized and must only be called from the main server thread during the plugin setup phase before the `BootEvent` is processed.
    - The core `runMigrations` method is highly concurrent. It orchestrates a massive fan-out of asynchronous tasks using `CompletableFuture` to load, process, and save individual world chunks in parallel. It uses thread-safe constructs like `AtomicInteger` for progress tracking.
    - **CRITICAL:** The module implements a critical synchronization pattern by disabling the `ChunkSavingSystems` before beginning its work and re-enabling it in a `finally` block. This acts as a system-wide lock on the chunk persistence layer, preventing race conditions with normal gameplay-driven chunk saving.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | MigrationModule | O(1) | Retrieves the global singleton instance. |
| register(id, migration) | void | O(1) | Registers a new migration factory. Must be called during plugin setup. |
| runMigrations() | void | O(N * M) | **High-Latency.** Initiates the full migration process for all configured worlds. N is the number of chunks, M is the number of migrations. |
| getChunkColumnMigrationSystem() | SystemType | O(1) | Returns the handle for the column-based migration system. |
| getChunkSectionMigrationSystem() | SystemType | O(1) | Returns the handle for the section-based migration system. |

## Integration Patterns

### Standard Usage
The primary integration point is registering a custom migration from another plugin. This is done by retrieving the singleton and calling `register` during your plugin's setup phase.

```java
// In your plugin's setup() method
MigrationModule migrationModule = MigrationModule.get();

// Register a new migration type named "v2-block-format"
// The provided lambda is a factory for your migration logic
migrationModule.register("v2-block-format", path -> new V2BlockFormatMigration(path));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new MigrationModule()`. The server's plugin loader is solely responsible for its lifecycle. Always use `MigrationModule.get()`.
- **Late Registration:** Do not call `register` after the server has finished its boot sequence. The migration logic is triggered by a `BootEvent`, and any registrations after this event will be ignored.
- **Manual Invocation:** Do not call `runMigrations` directly. This method is extremely destructive and assumes it has exclusive control over the world storage layer. It should only ever be triggered by the server's boot process in response to specific command-line flags. Manual invocation on a live server will lead to world corruption.

## Data Pipeline
The module executes a complex data transformation pipeline that reads from and writes to the primary world storage on disk.

> Flow:
> Command-Line Flags -> Server Bootstrap -> `BootEvent` -> **MigrationModule.runMigrations()** -> Disable ChunkSavingSystems -> Iterate World Chunk Indexes -> For each chunk: [ IChunkLoader (Disk Read) -> Deserialize to WorldChunk -> **Apply Migrations** -> Check for changes -> IChunkSaver (Disk Write) ] -> Re-enable ChunkSavingSystems -> Server Shutdown

