---
description: Architectural reference for UpdateAssetsCommand
---

# UpdateAssetsCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.git
**Type:** Transient

## Definition
```java
// Signature
public class UpdateAssetsCommand extends AbstractCommandCollection {
```

## Architecture & Concepts

The UpdateAssetsCommand class serves as a high-level administrative bridge between the server's command system and the host operating system's file system and Git tooling. It is not a game logic component but an infrastructure utility designed for server operators.

Architecturally, it functions as a **Command Group**, a container that aggregates related sub-commands under a single namespace, in this case *assets*. The primary class, UpdateAssetsCommand, does not contain any execution logic itself; it delegates all operations to its nested sub-commands: UpdateAssetsStatusCommand, UpdateAssetsResetCommand, and UpdateAssetsPullCommand.

Each sub-command inherits from AbstractAsyncCommand, a critical design choice that ensures all potentially long-running Git operations are executed **off the main server thread**. This prevents file I/O and external process execution from blocking the server's primary game loop, which would otherwise cause catastrophic performance degradation.

Execution is handled by shelling out to an external Git process via the Java ProcessBuilder API. This decouples the server from any specific Git library implementation and leverages the system's existing, robust Git installation. The command dynamically locates the asset repository, providing flexibility for different deployment environments.

A notable feature within the UpdateAssetsPullCommand is its ability to prefer a custom shell script (updateAssets.sh) over a standard *git pull*. This elevates the system from a simple Git wrapper to a more powerful deployment hook, allowing operators to define complex update procedures, such as clearing caches or running asset post-processing steps.

## Lifecycle & Ownership
-   **Creation:** A single instance of UpdateAssetsCommand is created by the server's central CommandRegistrar during the server bootstrap sequence. It is registered along with all other server commands.
-   **Scope:** The instance persists for the entire lifetime of the server session. As a command definition object, it is effectively stateless between executions.
-   **Destruction:** The object is de-referenced and becomes eligible for garbage collection when the server shuts down and the CommandRegistrar is cleared.

## Internal State & Concurrency
-   **State:** The UpdateAssetsCommand and its nested sub-commands are stateless. They do not cache data or maintain any state across multiple executions. All necessary information, such as the path to the asset repository, is resolved on-demand each time a command is invoked.
-   **Thread Safety:** The class is inherently thread-safe. The command's execution logic is encapsulated within the executeAsync method, which is invoked by the command system on a worker thread from a common thread pool. Each command execution spawns a new, isolated operating system process. This model guarantees that concurrent executions of the command will not interfere with each other.

## API Surface

The primary public contract of this class is its constructor, used for instantiation by the command system. Direct method invocation is not an intended use case.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| UpdateAssetsCommand() | Constructor | O(1) | Creates the command collection and registers its sub-commands. |

## Integration Patterns

### Standard Usage

This class is not intended for direct programmatic use. It is designed to be invoked by a server administrator through the server console.

```sh
# Check the current status of the asset repository
/assets status

# Discard local changes and revert to the last commit
/assets reset

# Pull the latest changes from the remote repository
/assets pull
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not create instances of this class using *new UpdateAssetsCommand()*. The server's command system is solely responsible for its lifecycle.
-   **Direct Invocation:** Never call the internal *executeAsync* method directly. Doing so bypasses the command system's permission checks, context setup, and thread management, which can lead to unpredictable behavior and server instability.
-   **Filesystem Assumptions:** Do not write code that assumes the asset directory is a Git repository. This command is an optional administrative tool; core game logic must not depend on its functionality or the underlying repository structure.

## Data Pipeline

The flow for this component is a command-and-control sequence, not a traditional data processing pipeline. It translates a user command into an OS process and streams the output back to the user.

> Flow:
> Server Console Input (`/assets pull`) -> Command Parser -> **UpdateAssetsCommand** (Router) -> `UpdateAssetsPullCommand.executeAsync` -> `ProcessBuilder` -> External `git` Process -> Process `stdout`/`stderr` Stream -> `CommandContext` -> Formatted Message -> Server Console Output

