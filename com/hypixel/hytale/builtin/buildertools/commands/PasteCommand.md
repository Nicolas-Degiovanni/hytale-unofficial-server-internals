---
description: Architectural reference for PasteCommand
---

# PasteCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class PasteCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The PasteCommand class is a command-line interface endpoint for the server's world editing functionality. It serves as a lightweight translator, converting a player-initiated command into a work item for the asynchronous `BuilderToolsPlugin` queue. Its primary architectural role is to decouple the command parsing and permission-checking layer from the resource-intensive logic of world modification.

This class does not perform any direct world manipulation. Instead, it gathers the necessary context—the executing player, their position, and any command arguments—and serializes this into a lambda expression. This work unit is then submitted to the `BuilderToolsPlugin` queue, ensuring that potentially long-running paste operations do not block the main server thread.

The class also demonstrates the "Usage Variant" pattern through its inner class, `PasteAtPositionCommand`. This allows a single root command, `paste`, to support multiple argument structures (e.g., pasting at the player's current location versus pasting at a specified coordinate) while maintaining a clean and organized implementation.

## Lifecycle & Ownership
- **Creation:** A single instance of PasteCommand is instantiated by the server's command registration system when the `BuilderToolsPlugin` is loaded. It is registered under the alias "paste".
- **Scope:** The PasteCommand object is a long-lived singleton that persists for the entire lifecycle of the plugin. However, the execution context provided to its `execute` method is transient and scoped only to a single command invocation.
- **Destruction:** The instance is de-registered and becomes eligible for garbage collection when the `BuilderToolsPlugin` is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** PasteCommand is entirely stateless. It holds no mutable fields related to its execution. All necessary information, such as the target player and world, is provided via method arguments during the `execute` call. The state of the clipboard itself is managed externally by the `BuilderToolsPlugin`.

- **Thread Safety:** This class is inherently thread-safe. As it is stateless, concurrent invocations of `execute` will not interfere with each other. The core operation, `BuilderToolsPlugin.addToQueue`, is a thread-safe method designed to accept work from multiple sources and process it on a dedicated worker thread. This design is critical for server performance and stability.

## API Surface
The public API is defined by its role as a command object within the server's command framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | protected void | O(1) | The entry point for the command system. Gathers context and queues the paste operation. The method itself is constant time, but the queued operation's complexity is proportional to the size of the clipboard data. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly via code. It is invoked by the server's command handler in response to a player executing the command in-game. The system retrieves the registered instance and calls its `execute` method.

*Player Input:*
```
/paste
```
*or*
```
/paste ~5 10 ~-2
```

*System-level Invocation (Conceptual):*
```java
// The server's command system finds the registered PasteCommand instance
// and invokes it with the context of the player who ran the command.
Command registeredCommand = commandRegistry.find("paste");
registeredCommand.execute(commandContext, ...);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new PasteCommand()`. The command must be resolved through the server's command registry to ensure permissions and context are correctly handled.
- **Assuming Synchronous Execution:** Do not write logic that assumes the world has been modified immediately after this command executes. The paste operation is asynchronous and will complete at a later time. Listen for world update events if you need to react to the completion of the paste.
- **Bypassing the Command:** Do not attempt to replicate this class's logic to add work to the `BuilderToolsPlugin` queue directly unless you are building a system that properly handles permissions and player context.

## Data Pipeline
The flow of data for a paste operation begins with player input and ends with a world state modification, with PasteCommand acting as the initial delegator.

> Flow:
> Player Chat Input (`/paste`) -> Server Command Parser -> **PasteCommand.execute** -> BuilderToolsPlugin.addToQueue -> Asynchronous World Edit Worker -> Clipboard Deserializer -> World.setBlock -> ChunkStore Update

