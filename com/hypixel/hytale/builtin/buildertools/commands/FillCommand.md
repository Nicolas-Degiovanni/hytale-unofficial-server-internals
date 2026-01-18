---
description: Architectural reference for FillCommand
---

# FillCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Registered Singleton

## Definition
```java
// Signature
public class FillCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The FillCommand class is a server-side command handler responsible for implementing the `/fill` chat command. It serves as a crucial entry point for players to perform large-scale world modification operations within the Builder Tools subsystem.

Architecturally, this class is not a direct world manipulator. Instead, it functions as a **validator and delegator**. Its primary responsibilities are:
1.  Defining the command's syntax, permissions, and aliases via the `CommandSystem` API.
2.  Parsing and validating the `BlockPattern` argument provided by the player.
3.  Verifying that the player has a valid selection region and the necessary permissions.
4.  **Asynchronously enqueueing** the actual fill operation with the `BuilderToolsPlugin`.

This delegation to a background work queue is the most critical design pattern in this class. It prevents large, computationally expensive fill operations from blocking the main server thread, which would otherwise cause severe server lag or crashes.

## Lifecycle & Ownership
- **Creation:** A single instance of FillCommand is instantiated by the server's command registry system during the bootstrap phase, typically when the `BuilderToolsPlugin` is loaded. It is not created on a per-request basis.
- **Scope:** The object is a long-lived singleton that persists for the entire server runtime. Its lifecycle is tied directly to the lifecycle of the command registry.
- **Destruction:** The instance is de-referenced and eligible for garbage collection only when the server shuts down or the parent plugin is unloaded, effectively unregistering the command.

## Internal State & Concurrency
- **State:** The FillCommand instance is effectively stateless with respect to individual command executions. It holds an immutable reference to its argument definition (`patternArg`), but all execution-specific data (like the player, world, and arguments) is passed into the `execute` method.
- **Thread Safety:** The FillCommand object itself is thread-safe due to its stateless design. The `execute` method is designed to be called exclusively from the server's main thread, which handles command processing.

**WARNING:** While the `execute` method is synchronous and thread-safe, the operation it triggers is not. The lambda passed to `BuilderToolsPlugin.addToQueue` will be executed on a separate worker thread. This handoff is a critical concurrency boundary.

## API Surface
The public contract is defined by its role as a command handler within the server framework. Direct invocation outside the command system is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | protected void | O(1) | Validates context and enqueues the fill operation. The method itself is constant time, but the enqueued task's complexity is proportional to the volume of the selected region. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is invoked automatically by the server's command processing system when a player executes the command in-game.

```
// Player-side action in chat
/fill hytale:stone
```

The server framework then routes this request to the registered FillCommand instance, populating the `execute` method's parameters from the game state.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new FillCommand()`. The object is meaningless unless registered with the server's `CommandSystem`, which handles its lifecycle.
- **Manual Execution:** Avoid calling the `execute` method directly. Doing so bypasses the framework's permission checks, argument parsing, and context setup, which can lead to `NullPointerException` or inconsistent game state.
- **Blocking Operations:** Do not modify this class to perform the fill operation directly within the `execute` method. All world modifications must be delegated to the `BuilderToolsPlugin` queue to maintain server performance.

## Data Pipeline
The flow of data for a fill operation begins with the player and terminates with a world state change, with FillCommand acting as a middleware component.

> Flow:
> Player Chat Input -> Network Packet -> Server Command Parser -> **FillCommand.execute()** -> Validation & Argument Parsing -> BuilderToolsPlugin Work Queue -> World Edit Worker Thread -> World Storage Modification -> Chunk Update Packets -> Client World Render

