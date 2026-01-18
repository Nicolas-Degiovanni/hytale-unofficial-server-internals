---
description: Architectural reference for InstanceEditLoadCommand
---

# InstanceEditLoadCommand

**Package:** com.hypixel.hytale.builtin.instances.command
**Type:** Transient

## Definition
```java
// Signature
public class InstanceEditLoadCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The InstanceEditLoadCommand serves as a high-level, user-facing entry point for a complex backend operation: loading a world instance from asset storage into a live, editable state. It is a classic implementation of the **Command Pattern**, designed to be discovered and executed by the server's central command processing system.

Architecturally, this class acts as a thin **Controller** or **Facade**. It does not contain the logic for loading worlds. Instead, it performs three critical functions:
1.  **Input Validation:** It defines and validates the required command arguments, in this case, the name of the instance to load, using an InstanceValidator.
2.  **Pre-condition Checks:** It performs crucial safety checks, most notably verifying that the server's base asset pack is mutable. This prevents catastrophic attempts to modify read-only production assets.
3.  **Orchestration:** It delegates the core, long-running task of loading the instance to the InstancesPlugin. Upon completion, it orchestrates the final step of teleporting the command's sender into the newly loaded world.

The extension of AbstractAsyncCommand is a deliberate and critical design choice. World loading is an I/O-heavy and computationally expensive operation. By returning a CompletableFuture, this command ensures that the main server thread is not blocked, maintaining server responsiveness.

## Lifecycle & Ownership
-   **Creation:** An instance of this command is created by the server's command registration framework during the server bootstrap or plugin loading phase. It is registered with the central command dispatcher under the name "load".
-   **Scope:** The object instance itself is effectively a singleton managed by the command framework. However, the *execution* of the command is transient and scoped to a single invocation. Each time a user runs the command, a new execution context is created, culminating in the completion of the returned CompletableFuture.
-   **Destruction:** The command object is destroyed when the server shuts down or when its parent plugin is unloaded, at which point it is deregistered from the command system and becomes eligible for garbage collection.

## Internal State & Concurrency
-   **State:** This class is fundamentally **stateless**. The field `instanceNameArg` is configuration, not mutable runtime state. It is initialized once in the constructor and is read-only thereafter. All state relevant to a specific execution is contained within the CommandContext object passed to the `executeAsync` method.

-   **Thread Safety:** This class is designed for a concurrent environment and is **conditionally thread-safe**.
    -   The `executeAsync` method is invoked on a server thread (e.g., the main game loop or a dedicated command thread).
    -   It immediately offloads the blocking work to `InstancesPlugin.loadInstanceAssetForEdit`, which executes on a background worker thread pool.
    -   The continuation logic within `thenAccept` is executed by the worker thread that completes the loading future.
    -   **WARNING:** Direct modification of game state from this continuation block is unsafe. The implementation correctly avoids this by scheduling a new task on the target world's dedicated thread via `playerWorld.execute`. This marshals the final operation (adding the Teleport component) back to the appropriate, thread-safe context, preventing race conditions.

## API Surface
The primary public contract is the `executeAsync` method, inherited and implemented from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context) | CompletableFuture<Void> | Asynchronous | Initiates the instance loading process. The initial call is O(1), but it triggers a long-running background task whose complexity depends on world size and disk I/O. |

## Integration Patterns

### Standard Usage
This command is not intended to be instantiated or invoked directly from application code. It is designed to be executed by a player or the server console through the chat or command line interface.

The framework handles the invocation internally, similar to this conceptual example:
```java
// CONCEPTUAL: How the command system invokes this class
CommandContext context = createFromPlayerInput("/instance edit load my_test_world");
InstanceEditLoadCommand command = commandRegistry.find("instance edit load");

// The framework calls executeAsync and manages the future
command.executeAsync(context);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new InstanceEditLoadCommand()`. The command system manages the lifecycle of registered commands. Direct instantiation bypasses registration and makes the command unusable.
-   **Blocking on the Future:** Calling `.get()` or `.join()` on the CompletableFuture returned by `executeAsync` from a main server thread will block that thread, likely causing the server to freeze until the world is fully loaded. This defeats the purpose of the asynchronous design.
-   **Cross-thread State Modification:** Modifying the player's entity or world state directly from the `thenAccept` callback is a severe concurrency violation. Always use the target world's `execute` method to schedule modifications, as shown in the implementation.

## Data Pipeline
This command initiates a control flow rather than a traditional data processing pipeline. The flow represents a sequence of events across multiple threads and systems.

> Flow:
> User Command Input -> Server Command Parser -> **InstanceEditLoadCommand.executeAsync** -> InstancesPlugin -> (Background Thread: Load World from Disk) -> CompletableFuture Completion -> (Callback on Background Thread) -> World.execute -> (Target World Thread: Add Teleport Component) -> Entity Component System -> Player Teleported

