---
description: Architectural reference for CutCommand
---

# CutCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class CutCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The CutCommand class is a server-side command handler responsible for processing the player-invoked `/cut` command. It serves as the primary user-facing entry point for the world modification feature that copies a region of the world to a player's clipboard and then removes the original.

Architecturally, this class is a thin-veneer controller. It does not contain the business logic for the cut operation itself. Instead, its responsibilities are:
1.  **Command Registration:** Declaring the command name (`cut`), required permissions, and argument structure to the server's command system.
2.  **Argument Parsing:** Interpreting flags provided by the user, such as `noEntities` or `empty`, and translating them into a bitmask of settings.
3.  **Context Validation:** Performing initial checks, such as verifying the player has a valid selection, before proceeding.
4.  **Task Delegation:** Asynchronously dispatching the core cut operation to the `BuilderToolsPlugin` work queue.

This delegation is a critical design pattern. World modification can be a resource-intensive task that should not block the main server thread. By queuing the operation via `BuilderToolsPlugin.addToQueue`, the command ensures server responsiveness. The actual logic for iterating through blocks, serializing entities, and modifying world storage is handled by the player's `BuilderState`.

The class also defines a nested `CutRegionCommand` variant. This allows the command system to handle two distinct usage patterns: `/cut` (which uses the player's current selection) and `/cut <coords>` (which uses explicit coordinates), while sharing much of the underlying implementation.

### Lifecycle & Ownership
-   **Creation:** A single instance of CutCommand is instantiated by the server's command registration system during server bootstrap. It is discovered and registered automatically.
-   **Scope:** The command object is a long-lived singleton for the duration of the server's execution. It is stateless and handles requests for all players.
-   **Destruction:** The instance is dereferenced and garbage collected only when the server is shutting down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** The CutCommand class is fundamentally stateless. Its member fields, such as `noEntitiesFlag`, are immutable definitions for the command argument parser. They do not store any per-player or per-execution data. All state related to a player's selection or clipboard is managed externally by `BuilderToolsPlugin.BuilderState`.

-   **Thread Safety:** This class is thread-safe. The `execute` method is invoked by the server's main thread. However, it immediately offloads the state-mutating work to the `BuilderToolsPlugin`'s per-player work queue. This design prevents race conditions and ensures that all world modifications for a given player are processed sequentially and off the main server tick loop. Direct access to this class from other threads is not a supported pattern.

## API Surface
The public contract is exclusively for interaction with the server's command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | void | O(1) | Validates context, parses arguments, and enqueues the cut operation. The method itself returns immediately. The enqueued task's complexity is proportional to the volume of the selected region. |

## Integration Patterns

### Standard Usage
This class is not designed for direct invocation by developers. The standard and only supported interaction is through a player executing the command in-game. The server's command dispatcher routes the request to the registered CutCommand instance.

```
// Player input in chat
/cut --noEntities
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new CutCommand()`. The command system manages the lifecycle of command objects. Manually creating an instance will result in a non-functional object that is not registered with the server.
-   **Manual Invocation:** Do not call the `execute` method directly. This bypasses critical infrastructure, including permission checks, argument parsing from the raw command string, and context setup provided by the command dispatcher.
-   **State Manipulation:** Do not attempt to read or modify the flag fields (e.g., `noEntitiesFlag`) at runtime. They are for definition purposes only and do not reflect the state of a specific command execution.

## Data Pipeline
The flow of data for a cut operation begins with the player and terminates with world storage modification, with CutCommand acting as an initial dispatcher.

> Flow:
> Player Chat Input -> Server Network Layer -> Command Dispatcher -> **CutCommand.execute()** -> `BuilderToolsPlugin.addToQueue()` -> Builder Tools Worker Thread -> `BuilderState.copyOrCut()` -> World Storage & Player Clipboard Update -> Network Packets to Clients

