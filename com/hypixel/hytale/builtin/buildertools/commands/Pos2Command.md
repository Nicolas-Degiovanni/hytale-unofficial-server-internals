---
description: Architectural reference for Pos2Command
---

# Pos2Command

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Command Object

## Definition
```java
// Signature
public class Pos2Command extends AbstractPlayerCommand {
```

## Architecture & Concepts
The Pos2Command class is a server-side command handler responsible for setting the second corner of a player's selection region for the Builder Tools system. It acts as a direct bridge between player chat input and the asynchronous task queue of the BuilderToolsPlugin.

This class follows the Command Pattern, encapsulating a specific user-initiated action into a self-contained object. Its primary architectural function is to parse user intent—either from explicit coordinates or the player's current location—and translate it into a deferred operation. It does not modify game state directly; instead, it delegates the actual state change to the centralized BuilderToolsPlugin, ensuring that all selection modifications are processed in a controlled and sequential manner.

By extending AbstractPlayerCommand, it integrates seamlessly into the server's core command processing pipeline, inheriting mechanisms for permission handling, argument parsing, and context injection.

### Lifecycle & Ownership
- **Creation:** A single instance of Pos2Command is created and registered by the server's Command System during the loading phase of the BuilderToolsPlugin. It is not instantiated on a per-request basis.
- **Scope:** The object is a singleton managed by the command registry. It persists for the entire lifecycle of the server or until the associated plugin is unloaded.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the Command System clears its registry, typically during plugin unload or server shutdown.

## Internal State & Concurrency
- **State:** The Pos2Command object is effectively immutable after construction. Its fields, such as xArg, yArg, and zArg, define the command's argument structure and do not change during runtime. The execute method is stateless, operating exclusively on the arguments provided by the command context for each invocation.
- **Thread Safety:** This class is thread-safe. The execute method is designed to be called from the server's main thread. Crucially, it offloads the stateful operation—modifying the player's selection—to the BuilderToolsPlugin queue. This design avoids race conditions and ensures that mutations to builder tool state are handled by the designated system, which is responsible for its own concurrency control.

## API Surface
The public contract is minimal, primarily consisting of the constructor for registration and the execute method for invocation by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Pos2Command() | constructor | O(1) | Defines the command name "pos2", its arguments, and required permission node "hytale.editor.selection.use". |
| execute(...) | void | O(1) | Parses arguments or player location and enqueues a selection update task. This operation is non-blocking. |

## Integration Patterns

### Standard Usage
This command is intended to be invoked by a player through the server's chat or console interface. The system automatically resolves the command, checks permissions, and invokes the execute method with the appropriate context.

```java
// This code is conceptual and represents how the Command System uses the class.
// A developer would not write this. A player triggers this via chat: /pos2 10 20 30

// Inside the Command System...
Command command = commandRegistry.get("pos2");
if (command.hasPermission(player)) {
    CommandContext context = buildContextForPlayer(player, args);
    command.execute(context, ...);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new Pos2Command()` in game logic. The command's lifecycle is exclusively managed by the server's command registry to ensure proper registration and singleton behavior.
- **Direct Invocation:** Avoid calling the `execute` method directly. Bypassing the command system means skipping critical steps like permission validation and argument parsing, which can lead to server instability or security vulnerabilities.
- **Synchronous Assumption:** Do not assume the player's selection is updated immediately after the command is run. The operation is queued and processed asynchronously. Any logic that depends on the new selection must be designed to handle this delay, likely by listening for a subsequent event.

## Data Pipeline
The flow of data for this command begins with player input and ends with a deferred state change in the Builder Tools system.

> Flow:
> Player Chat Input (`/pos2`) -> Network Packet -> Server Command System -> **Pos2Command.execute()** -> BuilderToolsPlugin Task Queue -> Selection State Update

