---
description: Architectural reference for PrefabPathEditCommand
---

# PrefabPathEditCommand

**Package:** com.hypixel.hytale.builtin.path.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabPathEditCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The PrefabPathEditCommand is a server-side command handler responsible for initiating a world path editing session for a player. It acts as a transactional bridge between a player's direct input (a chat command) and the modification of server-side player state.

This class follows the Command Pattern, encapsulating a specific user request—to edit a path—into a self-contained object. Its primary function is to resolve a target path, either via a direct UUID argument or by ray-casting to an entity the player is looking at, and then update the player's session data within the **BuilderToolsPlugin** to reflect that they are now actively editing this path.

It is a critical component for in-game world building and content creation, providing the entry point for designers and administrators to manipulate NPC patrol routes and other path-based systems.

## Lifecycle & Ownership
- **Creation:** A single instance of PrefabPathEditCommand is created by the server's CommandSystem during the plugin loading and command registration phase. It is not instantiated per-execution.

- **Scope:** The object instance persists for the entire server session. However, its operational scope is limited to the `execute` method call, which is invoked transactionally for each command usage. The state it modifies (player session data) is external and managed by other systems.

- **Destruction:** The instance is garbage collected when the server shuts down or when the parent plugin is unloaded, at which point the CommandSystem de-registers it.

## Internal State & Concurrency
- **State:** This class is effectively stateless. It contains a final field, `pathIdArg`, which defines a command argument structure. This definition is immutable and shared across all executions. All runtime state, such as the target player or the world's path data, is passed into the `execute` method as parameters.

- **Thread Safety:** This component is **not thread-safe**. The `execute` method is designed to be called exclusively from the main server thread (the game loop). All interactions with the EntityStore, World, and other game components are fundamentally single-threaded and must not be accessed from worker threads without explicit synchronization mechanisms, which are not present here.

## API Surface
The public contract is fulfilled by overriding the `execute` method from its parent, AbstractPlayerCommand.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Resolves a target path and activates an editing session for the command-issuing player. Throws no checked exceptions but sends failure messages to the player on invalid input. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation. It is designed to be registered with the server's command system, which then handles parsing player input and invoking the `execute` method with the correct context.

```java
// Example of registering the command within a plugin
// This is typically done once at server startup.

CommandSystem commandSystem = server.getCommandSystem();
commandSystem.register(new PrefabPathEditCommand());
```

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Do not manually instantiate and call the `execute` method. The `CommandContext`, `Store`, and `Ref` objects are complex and managed by the engine's command processing pipeline. Bypassing this will lead to a broken or inconsistent game state.

- **Stateful Implementation:** Do not add mutable member variables to this class. The single instance is shared and reused for all players executing the command. Storing per-player state here would create severe race conditions and data corruption.

## Data Pipeline
The command processes data in a clear, linear flow, translating a player's intent into a state change within a different system.

> Flow:
> Player Chat Input (`/path edit <uuid>`) -> Server Command Parser -> **PrefabPathEditCommand.execute()** -> TargetUtil (Optional Entity Lookup) -> WorldPathData (Read Path) -> BuilderToolsPlugin (Write Active Path) -> Player Session State Updated

