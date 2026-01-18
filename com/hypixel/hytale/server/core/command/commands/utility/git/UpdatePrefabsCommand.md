---
description: Architectural reference for UpdatePrefabsCommand
---

# UpdatePrefabsCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.git
**Type:** Service (Singleton-like)

## Definition
```java
// Signature
public class UpdatePrefabsCommand extends AbstractCommandCollection {
```
```java
// Core Logic Handler (Inner Class)
private abstract static class UpdatePrefabsGitCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts

The UpdatePrefabsCommand is a server-side administrative command that provides an interface for managing the server's prefab assets via an external Git repository. Architecturally, it is not a single command but a **Composite Command**, acting as a namespace for a suite of related Git operations (status, pull, push, etc.). It extends AbstractCommandCollection, a base class designed to group multiple subcommands under a single entry point.

The core logic is not implemented in the parent class but is delegated to a series of private static inner classes, all of which inherit from the abstract `UpdatePrefabsGitCommand`. This inner class is the key to the system's design and employs the **Template Method Pattern**. It defines a skeleton for executing an external process, handling asynchronous execution, and streaming output back to the command sender. Each concrete subclass, such as `UpdatePrefabsPullCommand`, simply provides the specific command-line arguments to be executed.

A critical design decision is the direct invocation of the system's **Git executable** via `java.lang.ProcessBuilder`. This component does not use a Java-native Git library like JGit. This choice simplifies the implementation and ensures that any local or system-wide Git configuration is respected, but it creates a hard dependency on the server's host environment.

Execution is fully asynchronous, inheriting from AbstractAsyncCommand. This ensures that potentially long-running network or disk I/O operations (like `git pull` or `git commit`) do not block the main server thread, which is essential for server stability and responsiveness.

## Lifecycle & Ownership

-   **Creation:** A single instance of UpdatePrefabsCommand is instantiated by the server's central `CommandSystem` during the server bootstrap phase. The command system scans for and registers all available command classes at this time.
-   **Scope:** The instance is registered and persists for the entire lifetime of the server session. It is effectively a singleton managed by the command registry.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency

-   **State:** The UpdatePrefabsCommand and its inner command classes are **stateless**. They do not maintain any mutable state across invocations. All required data, such as the path to the prefabs directory or the command sender's identity, is retrieved from external services (PrefabStore) or the provided CommandContext at the time of execution.
-   **Thread Safety:** This class is inherently thread-safe. As it is stateless, multiple administrators could execute commands concurrently without risk of interference. The `executeAsync` method is designed to be invoked by the command system's dedicated thread pool. Each execution is isolated, creating its own `ProcessBuilder` and I/O streams, preventing any cross-thread contamination. No explicit locking mechanisms are required.

## API Surface

The primary interface for this system is the server console. The programmatic API surface is defined by its interaction with the command framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context) | CompletableFuture<Void> | I/O Bound | (Inherited) Asynchronously executes a sequence of Git commands in a new process. Streams stdout/stderr back to the sender. |
| getCommands(senderDisplayName) | String[][] | O(1) | (Template Method) Implemented by subclasses to provide the specific Git commands and arguments to be executed. |

## Integration Patterns

### Standard Usage

This command is intended for server administrators to update game assets that are managed in a Git repository directly from the server console.

```sh
# Example of an administrator pulling the latest prefabs
/update prefabs pull

# Example of committing and pushing all local changes
/update prefabs all
```

The command system identifies the command and invokes the corresponding `executeAsync` method on a background thread.

```java
// Simplified conceptual view of framework invocation
CommandContext ctx = createFromConsoleInput("/update prefabs pull");
AbstractAsyncCommand command = commandRegistry.find("update", "prefabs", "pull");

// The framework runs this future on a worker thread pool
CompletableFuture<Void> future = command.executeAsync(ctx);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new UpdatePrefabsCommand()`. The command is useless unless registered with the server's command system, which handles its lifecycle.
-   **Blocking on the Future:** Never call `.get()` or `.join()` on the `CompletableFuture` returned by `executeAsync` from a primary server thread (e.g., the main game loop). This will block the thread and freeze the server until the Git process completes.
-   **Environment Misconfiguration:** The server must be run in an environment where the `git` command is available in the system's PATH. Failure to meet this prerequisite will result in an `IOException` for all subcommands, rendering the feature non-functional.

## Data Pipeline

The flow of data for this command involves the server's internal systems, the host operating system, and potentially a remote version control server.

> **Outbound Flow:**
> Server Console Input -> Command Parser -> **UpdatePrefabsGitCommand** -> ProcessBuilder -> Host OS Shell -> Git Executable -> Filesystem / Remote Git Server

> **Feedback Flow:**
> Git Executable (stdout/stderr) -> Process InputStream -> **UpdatePrefabsGitCommand** (Stream Reader) -> Message Translation Service -> CommandContext -> Server Console Output

