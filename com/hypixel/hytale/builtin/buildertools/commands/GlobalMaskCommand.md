---
description: Architectural reference for GlobalMaskCommand
---

# GlobalMaskCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Framework-Managed Handler

## Definition
```java
// Signature
public class GlobalMaskCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The GlobalMaskCommand class provides the server-side implementation for the `/gmask` chat command, a core feature of the Builder Tools plugin. It serves as the primary user-facing entry point for players to manage their global block modification mask. This mask restricts the effects of other builder tools, such as brushes and fills, to only operate on blocks that match the mask's criteria.

Architecturally, this class follows the **Composite Command Pattern**. The top-level GlobalMaskCommand acts as a router and default handler. It processes the base `/gmask` command to display the current mask. It then delegates more specific actions to nested private static subclasses:
*   **GlobalMaskSetCommand:** Handles setting a new mask.
*   **GlobalMaskClearCommand:** Handles removing the existing mask.

The most critical design aspect is its interaction with the **BuilderToolsPlugin**. This command does not directly mutate player or world state. Instead, it acts as a thin translation layer that validates input and permissions, then enqueues the requested state change operation onto the BuilderToolsPlugin's work queue via `BuilderToolsPlugin.addToQueue`. This design decouples the command processing logic from the game-state modification logic, ensuring all builder tool operations are executed synchronously on the main game thread, preventing race conditions and maintaining data integrity.

## Lifecycle & Ownership
- **Creation:** An instance of GlobalMaskCommand is created by the server's command registration system during the initialization phase of the BuilderToolsPlugin. The framework discovers and instantiates the command, adding it to a central command registry.
- **Scope:** The command object is a long-lived instance. It persists for the entire lifecycle of the server or until the owning plugin is disabled.
- **Destruction:** The object is eligible for garbage collection only when the BuilderToolsPlugin is unloaded or the server shuts down, at which point the command registry is cleared.

## Internal State & Concurrency
- **State:** The GlobalMaskCommand and its subclasses are **stateless**. They do not store any data related to a specific player or command execution between invocations. All necessary state, such as the player's current BlockMask, is managed by components within the Entity Component System and accessed via the BuilderToolsPlugin.

- **Thread Safety:** This class is **thread-safe by design**. The `execute` method may be called from various server threads (e.g., network I/O or a dedicated command processing thread). However, all state-mutating logic is encapsulated within a lambda and passed to the `BuilderToolsPlugin.addToQueue` method. This guarantees that the actual modification of the player's global mask occurs safely on the main server tick thread.

## API Surface
The primary API is the chat command interface exposed to players. Programmatic interaction is handled exclusively by the server's command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| /gmask | Command | O(1) | Displays the player's currently active global mask. |
| /gmask set <mask> | Command | O(1) | Sets the player's global mask. The mask argument is parsed by the framework. |
| /gmask clear | Command | O(1) | Clears the player's global mask, allowing tools to affect all blocks. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use by other plugins. It is invoked automatically by the server's command handler when a player executes the corresponding chat command. The system dispatches the command context to the `execute` method.

```java
// This code is conceptual and executed by the server's command system.
// A developer would not write this.

// 1. Player types "/gmask set stone" into chat.
// 2. Server parses the input and identifies GlobalMaskSetCommand as the handler.
// 3. The framework constructs a CommandContext and invokes the execute method.

// Inside GlobalMaskSetCommand.execute(context, ...):
BlockMask mask = this.maskArg.get(context); // "stone" is parsed into a BlockMask
BuilderToolsPlugin.addToQueue(playerComponent, playerRef, (r, s, c) -> s.setGlobalMask(mask, c));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new GlobalMaskCommand()`. The command system manages the lifecycle. Manual instantiation will result in a non-functional object that is not registered to handle any chat commands.
- **Bypassing the Queue:** Do not attempt to find the player's state component and call `setGlobalMask` directly from a separate thread. This circumvents the thread-safe queuing mechanism of the BuilderToolsPlugin and will lead to concurrency violations, data corruption, or server instability.

## Data Pipeline
The flow of data for a typical command execution demonstrates the separation of concerns between command parsing and state mutation.

> Flow:
> Player Chat Input -> Server Network Layer -> Command Dispatcher -> **GlobalMaskCommand.execute()** -> Argument Parsing -> **BuilderToolsPlugin.addToQueue()** -> Main Game Thread -> Queued Lambda Execution -> Player Component State Mutation

