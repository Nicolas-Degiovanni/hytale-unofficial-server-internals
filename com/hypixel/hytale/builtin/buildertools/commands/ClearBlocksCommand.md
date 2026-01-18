---
description: Architectural reference for ClearBlocksCommand
---

# ClearBlocksCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient Command Handler

## Definition
```java
// Signature
public class ClearBlocksCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The ClearBlocksCommand is a server-side, player-executable command responsible for removing blocks from the world within a specified region. It serves as a high-level user interface for a more complex, underlying world modification system, the BuilderToolsPlugin.

This command acts as a translator, converting player input—either from chat commands or from a pre-defined selection tool—into a formal, asynchronous world modification task. It does not perform the block removal itself. Instead, it serializes the player's intent into a task and submits it to the BuilderToolsPlugin queue. This decouples the immediate command execution from the potentially long-running and resource-intensive block manipulation, preventing the server from stalling on large operations.

The command supports two primary operational modes:
1.  **Selection-based Clearing:** When invoked without arguments (e.g., `/clear`), it uses the player's existing selection region, as managed by the BuilderToolsPlugin state.
2.  **Coordinate-based Clearing:** When invoked with two position arguments (e.g., `/clear ~-5 ~-5 ~-5 ~5 ~5 ~5`), it defines a new bounding box for the clear operation, independent of any prior selection.

Crucially, all operations are gated by permission checks, restricting its use to players in Creative mode.

## Lifecycle & Ownership
-   **Creation:** A single instance of ClearBlocksCommand is created by the server's command registration system during the bootstrap phase of the BuilderToolsPlugin. It is discovered and registered to handle the identifiers *clearBlocks* and *clear*.
-   **Scope:** The command object is a stateless handler that persists for the entire server session, or as long as its parent plugin is enabled. A single instance is reused for all invocations of the command by any player.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the BuilderToolsPlugin is disabled or the server shuts down, at which point the command is de-registered from the server's command dispatcher.

## Internal State & Concurrency
-   **State:** The ClearBlocksCommand instance is effectively immutable after construction. It holds no mutable state related to any specific player or command execution. All necessary context, such as the invoking player and world state, is provided as arguments to the *execute* method.

-   **Thread Safety:** This class is thread-safe by design due to its stateless nature. However, the *execute* method is exclusively invoked on the main server thread. It interacts with non-thread-safe game state components like the World and EntityStore.

    **WARNING:** Direct modification of world state within the *execute* method would be a severe concurrency violation. The class achieves safety by delegating all world-mutating work to the `BuilderToolsPlugin.addToQueue` method. This ensures that all block operations are serialized and processed correctly within the server's main tick loop or a dedicated worker, preventing race conditions and world corruption.

## API Surface
The public contract is defined by its role as a command handler, with the primary entry point being the `execute` method invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | void | O(N) | Parses command context and queues a world modification task. Complexity is O(1) for the command itself, but the resulting queued operation is O(N), where N is the number of blocks in the target region. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly via code. It is registered with the server's command system and invoked in-game by players with appropriate permissions. The system identifies the command from player chat and routes it to the registered handler.

The conceptual invocation by the command system looks like this:

```java
// PSEUDOCODE: How the command system invokes this handler
CommandContext context = CommandParser.parse("player typed /clear");
PlayerRef player = context.getSourcePlayer();
World world = player.getWorld();

// The system finds the registered ClearBlocksCommand instance
ClearBlocksCommand handler = commandRegistry.get("clear");

// The system invokes the handler with all necessary context
handler.execute(context, world.getEntityStore(), player.getRef(), player, world);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new ClearBlocksCommand()`. The command system manages the lifecycle of command handlers. Direct instantiation will result in a non-functional object that is not registered to receive any commands.
-   **Manual Invocation:** Do not call the `execute` method directly. Bypassing the command system's dispatcher will skip critical steps like permission validation, argument parsing, and context setup, leading to unpredictable behavior and likely NullPointerExceptions.
-   **Bypassing the Queue:** Do not attempt to modify the `ChunkStore` or world blocks directly from within this command. All world modifications **must** be routed through `BuilderToolsPlugin.addToQueue` to ensure operational integrity, support for undo/redo, and server stability.

## Data Pipeline
The flow of data for a clear operation begins with player input and ends with a change in the world state, with this command acting as a key intermediary.

> Flow:
> Player Chat Input (`/clear <args>`) -> Server Command Dispatcher -> **ClearBlocksCommand.execute()** -> BuilderToolsPlugin.addToQueue(task) -> World Edit Processor -> ChunkStore Block Update

