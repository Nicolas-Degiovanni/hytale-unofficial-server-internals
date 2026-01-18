---
description: Architectural reference for PrefabEditSaveCommand
---

# PrefabEditSaveCommand

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.commands
**Type:** Singleton

## Definition
```java
// Signature
public class PrefabEditSaveCommand extends AbstractAsyncPlayerCommand {
```

## Architecture & Concepts

The PrefabEditSaveCommand class is a user-facing entry point into the prefab serialization pipeline, exposed to players through the server's command system. It acts as a command-line controller that orchestrates the process of saving in-memory prefab edits to disk.

Architecturally, this class serves as a crucial bridge between the player's active editing session and the underlying persistence layer. It is not responsible for the low-level details of file I/O; instead, it delegates the core serialization work to the PrefabSaver utility. Its primary responsibilities are:

1.  **State Validation:** It queries the PrefabEditSessionManager to ensure the executing player is in a valid editing state.
2.  **Intent Parsing:** It interprets command-line flags (e.g., --saveAll, --noEntities) to configure the save operation.
3.  **Pre-condition Enforcement:** It performs critical checks before initiating a save, including selection boundary validation, read-only status confirmation, and security checks to prevent path traversal attacks.
4.  **Asynchronous Orchestration:** By extending AbstractAsyncPlayerCommand, it guarantees that potentially slow disk operations do not block the main server thread, preserving server performance. It uses CompletableFuture to manage the asynchronous workflow and provide feedback to the player upon completion.

This class embodies the **Facade** pattern, providing a simple, high-level command interface over the more complex prefab saving subsystem.

### Lifecycle & Ownership
-   **Creation:** A single instance of PrefabEditSaveCommand is created by the BuilderToolsPlugin during server startup or plugin initialization. It is then registered with the server's central command registry.
-   **Scope:** Application-scoped. The singleton instance persists for the entire lifetime of the server.
-   **Destruction:** The instance is de-registered from the command system and becomes eligible for garbage collection when the BuilderToolsPlugin is disabled or the server shuts down.

## Internal State & Concurrency
-   **State:** This class is effectively **stateless**. Its instance fields are immutable definitions for command arguments, configured once in the constructor. All state required for an operation is passed into the executeAsync method via the CommandContext or retrieved from external session managers. This design ensures that each command execution is isolated.

-   **Thread Safety:** The class is inherently **thread-safe**. Due to its stateless nature, concurrent executions of the command by different players do not interfere with one another. The asynchronous execution is managed by the Java CompletableFuture framework, which safely handles the operation on a worker thread pool, with callbacks scheduled back to the main server thread for player communication.

## API Surface

The primary public contract is its registration as a chat command. The core logic is encapsulated in the executeAsync method, which is invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context, store, ref, playerRef, world) | CompletableFuture<Void> | O(N) | Orchestrates the entire save process for one or more prefabs. Complexity is dominated by disk I/O, where N is the number of prefabs being saved. Failures are handled within the future, resulting in a feedback message to the player. |

## Integration Patterns

### Standard Usage

This class is not intended to be used directly in code. It is invoked by the server's command processing system when a player executes the corresponding chat command.

A developer interacts with this system by ensuring the BuilderToolsPlugin is loaded. A player would use it as follows in the game client:

```
// Saves the currently selected prefab
/ped save

// Saves the selected prefab, ignoring entities
/ped save --noEntities

// Saves all loaded prefabs, confirming overwrites of read-only files
/ped save --saveAll --confirm
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PrefabEditSaveCommand()`. An unregistered instance is non-functional as it will never be invoked by the command system.
-   **Direct Invocation:** Avoid calling the `executeAsync` method directly. Doing so bypasses the command system's essential setup, including permission checks, argument parsing, and context creation, leading to unpredictable behavior and NullPointerExceptions.
-   **Blocking on the Future:** On the main server thread, never call `.join()` or `.get()` on the CompletableFuture returned by `executeAsync`. This would negate the benefit of the asynchronous design and cause severe server lag or deadlocks.

## Data Pipeline

The flow of data and control for a save operation is a clear, multi-stage process orchestrated by this command.

> Flow:
> Player Command Input (`/ped save`) → Server Command Parser → **PrefabEditSaveCommand.executeAsync** → PrefabEditSessionManager (State Retrieval) → PrefabSaverSettings (Configuration) → PrefabSaver (Delegation) → Filesystem (Disk I/O) → CompletableFuture Callback → Player Feedback Message

