---
description: Architectural reference for WorldGenReloadCommand
---

# WorldGenReloadCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.worldgen
**Type:** Transient

## Definition
```java
// Signature
public class WorldGenReloadCommand extends AbstractAsyncWorldCommand {
```

## Architecture & Concepts
The **WorldGenReloadCommand** is a high-privilege administrative command responsible for hot-swapping the server's world generation system without requiring a full restart. It serves as the primary interface for developers and server operators to apply new world generation logic to a running world instance.

Architecturally, this command acts as an orchestrator for a complex, multi-stage, and potentially destructive operation. Its design is centered around safety and preventing server instability. The most critical architectural feature is its use of a static, global lock, **IS_RUNNING**, which ensures that only one world regeneration process can occur at any given time across the entire server. This singleton operational model is essential to prevent catastrophic race conditions and data corruption that would arise from concurrent modification of world generator settings and chunk data.

The command's logic is bifurcated into two main paths:
1.  **Configuration Reload:** A lightweight process that re-initializes the **IWorldGen** and **IWorldMap** providers from the server's configuration files and injects them into the active **World** object. This affects all newly generated chunks.
2.  **Chunk Purge & Regeneration:** An optional, heavyweight process triggered by the *clear* flag. This path orchestrates the complete deletion of all persisted chunk data from storage, followed by the regeneration of any chunks currently loaded in memory. This is an I/O-intensive, asynchronous operation that leverages the world's dedicated executor to avoid blocking the main server thread.

## Lifecycle & Ownership
-   **Creation:** A prototype instance of **WorldGenReloadCommand** is instantiated by the server's command registration system during the initial server bootstrap phase.
-   **Scope:** The prototype instance is a singleton that persists for the entire server session. The execution context of the command, however, is transient and tied to a specific invocation by a user.
-   **Destruction:** The instance is dereferenced and garbage collected when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** An instance of this class holds minimal state, primarily the definition for its command arguments like **clearArg**. The most significant state it manages is the static **IS_RUNNING** AtomicBoolean. This flag is external to any single instance and acts as a global mutex for the world regeneration process. It is highly mutable and volatile.

-   **Thread Safety:** The command's execution is fundamentally thread-safe due to its architectural design. The static **IS_RUNNING** lock prevents concurrent executions. Any attempt to run the command while it is already in progress will be rejected immediately. Furthermore, the most performance-intensive parts of the operation, such as deleting and regenerating chunks, are delegated to the world's asynchronous task scheduler. This ensures that all interactions with world components and chunk storage are performed on the correct thread, preventing concurrency violations within the game engine itself.

    **WARNING:** Direct external modification of the **IS_RUNNING** static field will break the concurrency model and can lead to severe server instability and world corruption.

## API Surface
The primary entry point is the **executeAsync** method, inherited from its parent class and invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context, world) | CompletableFuture<Void> | O(N) | Orchestrates the entire reload and optional clear process. The operation is globally locked and fully asynchronous. Complexity is O(1) for a simple reload, but becomes O(N) when the *clear* flag is used, where N is the total number of persisted chunks in the world. |

## Integration Patterns

### Standard Usage
This class is a command and is not intended to be used directly in code. A server administrator invokes it through the server console or as an in-game command.

**Example 1: Reloading the generator configuration**
This command updates the world generator for all newly generated chunks without affecting existing terrain.
```sh
# Server Console Input
/worldgen reload
```

**Example 2: Full world regeneration**
This command reloads the generator and additionally deletes all existing chunks, forcing the world to regenerate from scratch using the new generator as players explore.
```sh
# Server Console Input
/worldgen reload --clear
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not create an instance using `new WorldGenReloadCommand()`. The command system manages the lifecycle of command objects. Direct instantiation bypasses the registration and permission systems.
-   **Bypassing the Command System:** Do not invoke the **executeAsync** method directly. This can bypass important contextual setup and error handling provided by the command dispatcher.
-   **Calling Private Methods:** The **clearChunks** method is a private utility with a strict operational contract. It assumes that chunk saving has been disabled and the global lock is held. Calling it directly from external code will lead to world corruption.

## Data Pipeline
The command initiates a complex data and control flow, especially when clearing chunks.

> **Flow (Command Invocation):**
> Server Console Input (`/worldgen reload`) -> Command Parser -> Command Dispatcher -> **WorldGenReloadCommand.executeAsync** -> WorldConfig & WorldMapManager Update

> **Flow (Chunk Clearing Sub-Process):**
> **WorldGenReloadCommand** -> ChunkStore (Disable Saving) -> IChunkSaver (Request All Chunk Indexes) -> IChunkSaver (Issue Parallel Deletion for each Chunk) -> ChunkStore (Regenerate In-Memory Chunks) -> ChunkStore (Re-enable Saving) -> User Feedback Messages

