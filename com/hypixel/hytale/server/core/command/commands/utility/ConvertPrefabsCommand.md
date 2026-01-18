---
description: Architectural reference for ConvertPrefabsCommand
---

# ConvertPrefabsCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility
**Type:** Transient

## Definition
```java
// Signature
public class ConvertPrefabsCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The ConvertPrefabsCommand is a server-side administrative utility designed for offline data migration of prefab files. Its primary function is to read Hytale prefab files (`.prefab.json`), update their internal data structures to conform to current game data (such as block and entity IDs), and overwrite the original files.

Architecturally, this command serves as a powerful developer tool, not as part of the core gameplay loop. It operates by creating a temporary, isolated, in-memory world instance. This sandboxed world is not used for gameplay but acts as a data context provider, enabling the command to resolve block and entity identifiers against a valid game state. This is a critical design choice, as it allows prefab conversion to occur on a running server without interfering with live game worlds, while still having access to the necessary registries and data stores for validation.

The command is implemented as an `AbstractAsyncCommand` to prevent long-running file I/O operations from blocking the main server thread. It processes files in throttled batches to manage system load, demonstrating a robust design for handling potentially thousands of files in large projects.

## Lifecycle & Ownership
-   **Creation:** A single instance of ConvertPrefabsCommand is created by the server's command registration system during application bootstrap. It is registered and held for the server's lifetime.
-   **Scope:** The command object itself is a long-lived singleton managed by the command system. However, each execution of the command is a distinct, short-lived operation. The `CompletableFuture` returned by `executeAsync` and the temporary world created within it exist only for the duration of a single command invocation.
-   **Destruction:** The command instance is discarded when the server shuts down. Critically, the temporary world created for conversion is explicitly destroyed at the end of each execution via `Universe.get().removeWorld()`, ensuring no memory or resource leaks.

## Internal State & Concurrency
-   **State:** The class instance holds immutable configuration for its arguments (e.g., `blocksFlag`, `pathArg`). It is otherwise stateless between executions. All state related to a specific conversion task, such as the lists of failed or skipped files, is created and managed within the scope of the `executeAsync` method.
-   **Thread Safety:** This class is designed for asynchronous execution and is not thread-safe if its methods were to be invoked concurrently on the same instance. The command system ensures that `executeAsync` is called in a controlled manner. The core logic is executed off the main server thread via `CompletableFuture`.

    **WARNING:** The method passes standard `ObjectArrayList` instances into asynchronous processing chains. While the `CompletableFuture` API orchestrates the execution flow to prevent simultaneous writes, this pattern is fragile. Any modification to the asynchronous chain that could introduce parallel writes to these lists would lead to race conditions.

## API Surface
The public contract is fulfilled by extending `AbstractAsyncCommand` and is invoked by the server's command system, not by direct method calls.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context) | CompletableFuture<Void> | O(N) | Initiates the asynchronous prefab conversion process. Complexity is O(N) where N is the number of prefab files found. The operation is heavily I/O bound. |

## Integration Patterns

### Standard Usage
This command is intended to be run from the server console by an administrator or via an automated script. It is not designed to be integrated into other game logic systems.

```java
// Example invocation from the server console
// This converts all prefabs in the 'asset' store, processing both blocks and entities,
// and overwriting files with the updated data.
/convertprefabs store=asset blocks=true entities=true destructive=true
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance of this class manually. The command system manages its lifecycle. `new ConvertPrefabsCommand()` will not work as it will not be registered with the server.
-   **Blocking Execution:** Do not call `.join()` or `.get()` on the `CompletableFuture` returned by `executeAsync` from a synchronous context like the main server thread. This will freeze the server.
-   **Running Without Backups:** The `destructive` flag modifies files in-place. Executing this command without a version control system (like Git) or a file backup is extremely risky and can lead to irreversible data corruption if the process fails or is interrupted.
-   **Ignoring Output:** The command reports skipped and failed files to the console. Ignoring these messages may leave prefabs in a broken or outdated state.

## Data Pipeline
The command orchestrates a complex data transformation pipeline that involves file system I/O, data deserialization, in-memory world simulation, and data reserialization.

> Flow:
> Console Command -> Command System Parser -> **ConvertPrefabsCommand.executeAsync**
> 1.  **Argument Parsing:** Flags (`--blocks`, `--destructive`) and arguments (`store=`, `path=`) are parsed from the `CommandContext`.
> 2.  **Path Discovery:** The `PrefabStore` service is queried to resolve symbolic store names (`asset`, `server`) into concrete file system paths.
> 3.  **Sandbox Creation:** A temporary, lightweight `World` is created in the `Universe`. This world uses `DummyWorldGenProvider` and `EmptyChunkStorageProvider`, ensuring it has no physical footprint and loads instantly. Its sole purpose is to provide access to valid `EntityStore` and `ChunkStore` instances for data resolution.
> 4.  **File Scanning:** The target directory is recursively scanned for all files ending in `.prefab.json`.
> 5.  **Batch Processing:** The list of files is divided into batches of 10. A `CompletableFuture` chain processes these batches sequentially with a small delay between them to avoid overwhelming the system. Within a batch, prefabs are processed in parallel.
> 6.  **Single Prefab Conversion:**
>     -   **Read:** `BsonUtil` reads the `.prefab.json` file from disk into a BsonDocument.
>     -   **Deserialize:** `SelectionPrefabSerializer` converts the BsonDocument into a `BlockSelection` Java object.
>     -   **Transform:** The `BlockSelection` object is mutated. This can include fixing filler blocks, relativizing coordinates, or, most importantly, reserializing entity and block data against the temporary world's stores.
>     -   **Serialize:** The modified `BlockSelection` object is serialized back into a BsonDocument.
>     -   **Write:** `BsonUtil` writes the new BsonDocument back to the original file path, overwriting it.
> 7.  **Cleanup:** Upon completion of all batches, the temporary `World` is removed from the `Universe`.
> 8.  **Reporting:** A summary of skipped and failed files is formatted and sent as messages to the command's source (e.g., the console).

