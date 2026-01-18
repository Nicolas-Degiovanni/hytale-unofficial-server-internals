---
description: Architectural reference for WallsCommand
---

# WallsCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class WallsCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The WallsCommand class is a server-side command processor responsible for translating a player's chat input into a deferred world modification task. It serves as a high-level user interface for a complex builder tool operation, specifically the creation of walls, floors, and roofs within a player-defined selection.

Architecturally, this class is a critical component of the **Command Pattern** implementation within the server. Its primary responsibility is not to perform the world edit itself, but to:
1.  Define the command's syntax, aliases, and required permissions.
2.  Parse and validate player-provided arguments (block type, thickness, flags).
3.  Perform pre-condition checks, such as verifying a valid player selection exists.
4.  Package the validated parameters into a functional task.
5.  Delegate the task to the BuilderToolsPlugin's asynchronous processing queue.

This delegation is the most significant design feature. By handing off the computationally expensive world modification to a separate system, the WallsCommand ensures that the primary server thread is not blocked, maintaining server performance and responsiveness.

## Lifecycle & Ownership
-   **Creation:** A single instance of WallsCommand is instantiated by the server's command registration system during the bootstrap phase of the BuilderToolsPlugin. It is discovered and registered based on its class definition.
-   **Scope:** The command definition object persists for the entire server session, acting as a stateless template for handling all `/walls` command invocations.
-   **Destruction:** The object is dereferenced and eligible for garbage collection when the BuilderToolsPlugin is unloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** The WallsCommand object is effectively **immutable** after its constructor completes. Its internal fields, which define the command's arguments and metadata, are configured once and are not modified during its lifecycle. All state related to a specific execution is passed into the `execute` method via the CommandContext.

-   **Thread Safety:** This class is thread-safe. The immutable nature of the command definition object allows it to be safely referenced from any thread. The `execute` method is designed to be called by the server's main command processing thread. Its handoff to the `BuilderToolsPlugin.addToQueue` method is a thread-safe operation, ensuring a clean transition of work to a background processing system.

## API Surface
The public contract is primarily defined by its inheritance from AbstractPlayerCommand and the implementation of the `execute` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | void | O(1) | Validates context and enqueues a world modification task. This method is non-blocking and its own complexity is constant time. The complexity of the enqueued task is significant but handled asynchronously. |

## Integration Patterns

### Standard Usage
A developer does not invoke this class directly. The server's command handler invokes the `execute` method in response to player input. The key integration pattern demonstrated within this class is the delegation of work to a managed, asynchronous queue.

```java
// The core logic inside execute() demonstrates the correct pattern:
// Do not perform the work here. Package the work as a lambda
// and submit it to a dedicated service.

BuilderToolsPlugin.addToQueue(
   playerComponent,
   playerRef,
   (r, s, componentAccessor) -> s.walls(r, pattern, this.thicknessArg.get(context), roof, floor, componentAccessor)
);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance with `new WallsCommand()`. The command system manages the lifecycle. A manually created instance will not be registered and will be non-functional.
-   **Blocking Operations:** The `execute` method must never contain long-running or blocking code, such as direct world manipulation loops. Doing so would violate the core architectural principle of the command system and cause severe server lag. Always delegate to an asynchronous service like the BuilderToolsPlugin queue.
-   **Stateful Implementation:** Do not add mutable instance fields to a command class to store state between executions. Commands must be stateless, as a single instance handles requests from all players.

## Data Pipeline
The flow of data for a successful command execution follows a clear, decoupled path from user input to world state change.

> Flow:
> Player Chat Input (`/walls stone --floor`) -> Server Network Layer -> Command Parser & Dispatcher -> **WallsCommand.execute** -> BuilderToolsPlugin Task Queue -> World Edit Processor -> World Storage & Replication

