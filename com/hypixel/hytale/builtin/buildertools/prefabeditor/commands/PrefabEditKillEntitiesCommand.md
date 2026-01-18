---
description: Architectural reference for PrefabEditKillEntitiesCommand
---

# PrefabEditKillEntitiesCommand

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabEditKillEntitiesCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The PrefabEditKillEntitiesCommand is a server-side command handler that implements the Command Pattern. It integrates directly into the server's command processing system and is responsible for translating a player-initiated action into a world-mutating operation.

This class serves as a specific endpoint within the Builder Tools plugin, exposing functionality of the Prefab Editor to players via a chat command. Its core architectural role is to act as a controller that orchestrates interactions between several distinct systems:

1.  **PrefabEditSessionManager:** It queries this manager to retrieve the stateful context of the player's current editing session.
2.  **EntityStore:** It issues destructive commands to the world's primary entity database.
3.  **TargetUtil:** It uses this spatial query utility to identify which entities fall within the command's operational scope.

This command is strictly confined to the server environment and has no client-side counterpart. Its logic is executed entirely within the server's main update loop upon invocation.

### Lifecycle & Ownership
-   **Creation:** A single instance of this class is instantiated by the server's command registration system when the BuilderToolsPlugin is loaded. The system discovers all classes extending AbstractPlayerCommand and manages their lifecycle.
-   **Scope:** The object instance is a stateless singleton that persists for the entire server lifetime. The operational state is not stored in the command itself but is passed into the execute method for each invocation.
-   **Destruction:** The instance is dereferenced and garbage collected when the server shuts down or the parent plugin is unloaded.

## Internal State & Concurrency
-   **State:** This class is fundamentally stateless. Its fields are static final Message constants, which are immutable. All data required for an operation is provided as arguments to the execute method, ensuring that each execution is independent and idempotent given the same world state.
-   **Thread Safety:** The class instance is inherently thread-safe due to its stateless design. However, the operations performed within the execute method, specifically querying and removing entities from the EntityStore, are **not** guaranteed to be thread-safe. The command system's execution model ensures that this method is invoked on the primary world thread, preventing race conditions with other game logic.

    **Warning:** Any attempt to invoke the execute method from an asynchronous thread will lead to world state corruption and server instability.

## API Surface
The public contract is defined by its superclass, AbstractPlayerCommand. The primary entry point is the overridden execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(N) | Executes the entity removal logic. Complexity is dependent on the spatial query performance of TargetUtil, where N is the number of entities in the queried region. Throws exceptions on invalid world state. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically discovered and executed by the server's command system in response to player input.

A player in-game triggers this command's execution by typing its registered name into the chat console.

```
# Player enters the following command in chat
/prefab edit kill
```

The server command parser identifies the command, resolves it to the registered PrefabEditKillEntitiesCommand instance, and invokes its execute method with the appropriate context.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PrefabEditKillEntitiesCommand()`. The command system is responsible for the lifecycle of command objects. Manually creating an instance will result in an object that is not registered with the server and is therefore non-functional.
-   **Manual Invocation:** Avoid calling the execute method directly. Bypassing the command system's dispatcher means you circumvent critical infrastructure, including permission checks, argument validation, and thread safety guarantees.

## Data Pipeline
The flow of data and control for this command is linear and initiated by the player.

> Flow:
> Player Chat Input -> Server Network Listener -> Command Parser -> **PrefabEditKillEntitiesCommand.execute()** -> PrefabEditSessionManager (Read State) -> TargetUtil (Spatial Query) -> EntityStore (Write/Delete) -> CommandContext (Send Response) -> Player Chat UI

