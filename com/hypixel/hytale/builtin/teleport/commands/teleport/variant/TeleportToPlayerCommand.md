---
description: Architectural reference for TeleportToPlayerCommand
---

# TeleportToPlayerCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.teleport.variant
**Type:** Singleton

## Definition
```java
// Signature
public class TeleportToPlayerCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The TeleportToPlayerCommand class is a server-side command handler responsible for executing the logic when a player teleports to another player. It is a specific "variant" of the broader teleport command system.

Architecturally, this class acts as a translator between the user-facing Command System and the server's underlying Entity Component System (ECS). It does not perform the teleportation directly. Instead, its primary function is to gather the required state from the source and target entities (players) and then create and attach a **Teleport** component to the source entity.

A separate, dedicated system (e.g., a TeleportSystem) is responsible for processing entities with a Teleport component on a subsequent server tick. This decouples the command's intent from its implementation, adhering to a pure ECS pattern where systems operate on components. This class is the *initiator* of the teleport action, not the executor.

## Lifecycle & Ownership
- **Creation:** A single instance is created by the command registration system during server bootstrap. It is registered as a handler for a specific command signature (e.g., `/teleport <player>`).
- **Scope:** The instance is a singleton that persists for the entire server session. It is stateless with respect to its execution, with all necessary context passed into the `execute` method.
- **Destruction:** The object is garbage collected when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its only field, `targetPlayerArg`, is an immutable definition for a command argument, configured once during construction. All state required for execution is provided via the `execute` method parameters.
- **Thread Safety:** This class is **not thread-safe** for direct concurrent access. However, it is designed to operate safely within the engine's threading model. The `execute` method is invoked by the command system, potentially on a separate thread pool. All interactions with the game world (reading or writing component data) are correctly marshaled onto the appropriate world's main thread via the `world.execute(...)` and `targetWorld.execute(...)` lambda submissions. This is a critical mechanism to prevent race conditions and ensure all entity modifications are serialized within the game loop.

**WARNING:** Any modification of this class must strictly adhere to the `world.execute` pattern for all world state interactions. Direct access to an entity's components from the command execution thread will lead to severe concurrency bugs and server instability.

## API Surface
The primary contract is the `execute` method, which is invoked by the command system framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | protected void | O(1) | Gathers source and target player data and schedules the addition of a Teleport component to the source player. Complexity is O(1) as it involves a few component lookups and scheduling a task. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by developers. It is invoked automatically by the server's command processing system when a player executes the corresponding chat command.

```
// Player input in chat client
/teleport OtherPlayerUsername
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new TeleportToPlayerCommand()`. The command system manages the lifecycle of command handlers.
- **Direct Invocation:** Do not call the `execute` method directly. Bypassing the command system will skip critical permission checks, argument parsing, and context setup.
- **Cross-Thread World Modification:** Do not access or modify components from the `store` or `targetStore` outside of a `world.execute()` block. This will break thread-safety guarantees.

## Data Pipeline
This command handler sits at the intersection of the command system and the game logic, initiating a data flow that results in a player's position being changed.

> Flow:
> Player Chat Input -> Command System Parser -> **TeleportToPlayerCommand.execute()** -> World Scheduler (`world.execute`) -> Add **Teleport** Component -> Teleport System (Next Tick) -> Update **TransformComponent**

