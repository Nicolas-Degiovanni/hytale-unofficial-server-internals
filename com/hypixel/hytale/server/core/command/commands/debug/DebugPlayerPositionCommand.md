---
description: Architectural reference for DebugPlayerPositionCommand
---

# DebugPlayerPositionCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug
**Type:** Command Handler

## Definition
```java
// Signature
public class DebugPlayerPositionCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The DebugPlayerPositionCommand is a server-side diagnostic tool that implements the Command software design pattern. It is registered with the server's central Command System and is invoked when a privileged user executes the corresponding chat command.

Its primary architectural role is to act as a read-only probe into the server's Entity Component System (ECS). Upon execution, it queries the state of several core components associated with the executing player's entity, such as TransformComponent and HeadRotation. This provides a precise, real-time snapshot of the player's spatial data as understood by the server simulation.

In addition to reporting data, the command also demonstrates a write-path into the game world by using the DebugUtils service to spawn a temporary visual marker. This confirms that the reported position aligns with the world's coordinate system. This class is a quintessential example of a debug-specific system that directly interfaces with core game state for diagnostic purposes, without being part of the primary gameplay loop.

## Lifecycle & Ownership
- **Creation:** A single instance of DebugPlayerPositionCommand is created by the Command System during server initialization. The system scans for classes that extend command base types and instantiates them for registration.
- **Scope:** The object is a stateless singleton that persists for the entire lifetime of the server. It holds no per-execution state in its member fields.
- **Destruction:** The instance is dereferenced and garbage collected only when the server shuts down and the Command System is dismantled.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. All data required for an operation is passed as arguments to the execute method. It does not cache data or maintain any state across multiple invocations. This design is critical for ensuring that commands are re-entrant and do not cause side effects.

- **Thread Safety:** **This class is not thread-safe and must only be invoked from the main server thread.** It directly reads from the EntityStore and writes to the World. These core data structures are not designed for concurrent access, and all interactions are expected to be serialized within the server's main tick loop. Calling the execute method from an external thread will lead to data corruption, race conditions, and server instability.

## API Surface
The public contract is limited to the constructor, used by the framework, and the execute method, which contains the command's logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Executes the command logic. Reads multiple components from the EntityStore, sends a formatted message to the player, and spawns a debug sphere in the world. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. It is a handler that is automatically discovered and invoked by the server's command processing pipeline in response to a player's chat input. The system provides the necessary context, entity store access, and player reference at runtime.

A conceptual view of the invocation flow:

```java
// PSEUDOCODE: Represents the Command System's dispatch loop
void onPlayerChatCommand(PlayerRef player, String commandName, String[] args) {
    AbstractCommand command = commandRegistry.find(commandName);
    if (command instanceof DebugPlayerPositionCommand) {
        // The system builds the context and retrieves the necessary state
        CommandContext context = buildContextForPlayer(player, args);
        Store<EntityStore> store = world.getEntityStore();
        Ref<EntityStore> entityRef = player.getEntityRef();
        
        // The command is executed with all dependencies injected
        command.execute(context, store, entityRef, player, world);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new DebugPlayerPositionCommand()`. The command must be managed by the server's Command System to function correctly. Manual instantiation will result in a non-functional object that is not registered to handle any chat commands.
- **Manual Invocation:** Do not call the `execute` method directly. Bypassing the Command System's dispatch logic will skip critical steps like permission checks, argument parsing, and context setup, leading to unpredictable behavior and potential NullPointerExceptions.
- **Stateful Modifications:** Do not modify this class to store state in member variables. Commands must remain stateless to ensure they behave predictably and do not introduce side effects between different executions or for different players.

## Data Pipeline
The command acts as a bridge, transforming a user request into a read operation on the ECS and subsequent write operations to the player's chat and the game world.

> Flow:
> Player Chat Input (`/debugplayerposition`) -> Network Layer -> Command Parser -> **DebugPlayerPositionCommand.execute** -> Reads from EntityStore (TransformComponent, etc.) -> Writes to PlayerRef (Message) AND World (DebugUtils.addSphere)

