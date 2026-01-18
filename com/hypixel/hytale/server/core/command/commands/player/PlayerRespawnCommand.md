---
description: Architectural reference for PlayerRespawnCommand
---

# PlayerRespawnCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player
**Type:** Singleton Command Handler

## Definition
```java
// Signature
public class PlayerRespawnCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The PlayerRespawnCommand class is a server-side command handler that bridges user input to direct manipulation of a player's entity state. It serves a single, critical function: to force a player to respawn by removing their DeathComponent from the Entity Component System (ECS).

This class exemplifies a common command system pattern by implementing two distinct execution paths:
1.  **Self-Respawn:** The primary class handles the `/respawn` command without arguments, targeting the player who executed the command. This logic is simpler and assumes the execution context is already on the correct world thread.
2.  **Other-Player Respawn:** A nested private class, PlayerRespawnOtherCommand, handles the `/respawn <player>` variant. This path is more complex, as it must safely locate the target player's entity reference and schedule the state mutation on the appropriate world thread to prevent concurrency issues.

This command operates directly on the core game state via the ECS Store and Ref objects, making it a powerful and sensitive piece of server logic.

## Lifecycle & Ownership
-   **Creation:** A single instance of PlayerRespawnCommand is instantiated by the server's CommandRegistry during the server bootstrap phase. The system scans for command classes and registers them for runtime lookup.
-   **Scope:** The command object is a long-lived singleton that persists for the entire server session. It acts as a stateless template for handling all `/respawn` command invocations.
-   **Destruction:** The instance is dereferenced and eligible for garbage collection only when the server shuts down and the CommandRegistry is cleared.

## Internal State & Concurrency
-   **State:** This class is designed to be **stateless and immutable**. Its fields are static final Message objects, which are constant. All mutable state required for execution, such as the target player and world, is provided externally via the CommandContext and method parameters.

-   **Thread Safety:** The class is thread-safe by design, employing a critical cross-thread scheduling pattern.
    -   The base command `execute` method, which extends AbstractPlayerCommand, is guaranteed by the command system to be invoked on the command-issuing player's world thread. Direct state mutation is safe in this context.
    -   The nested PlayerRespawnOtherCommand's `executeSync` method may be called from any thread. It correctly retrieves the target player's World and uses **world.execute(...)** to enqueue the state-mutating logic (removing the DeathComponent) on the target world's dedicated thread. This is the mandatory pattern for safely modifying entity state from an external context.

    **Warning:** Failure to use the world's execution queue for cross-entity modifications will lead to severe race conditions, data corruption, and server instability.

## API Surface
The public API is intended for consumption by the command system, not for direct developer invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PlayerRespawnCommand() | constructor | O(1) | Instantiates the command and registers its "other player" usage variant. |
| execute(...) | protected void | O(1) | Framework-invoked method. Removes the DeathComponent from the command issuer. |
| executeSync(...) | protected void | O(1) | Framework-invoked method for the nested class. Schedules the removal of the DeathComponent for a specified target player. |

## Integration Patterns

### Standard Usage
A developer or user does not interact with this class directly. The system is invoked via the in-game chat or server console. The command framework handles parsing, routing, and execution.

*Conceptual invocation by a player:*
```
/respawn
```

*Conceptual invocation by an administrator or system:*
```
/respawn Notch
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PlayerRespawnCommand()`. The command system relies on the single, registered instance for permission checks, argument parsing, and lifecycle management.
-   **Manual Execution:** Never call the `execute` or `executeSync` methods directly. This bypasses the entire command processing pipeline, including critical safety and threading guarantees.
-   **Unsafe State Mutation:** Do not attempt to replicate this class's logic by directly accessing an EntityStore from another thread. Any modification to an entity's components **must** be scheduled on that entity's world thread.

    *Incorrect Implementation:*
    ```java
    // DANGEROUS: Modifying state from the wrong thread
    Ref<EntityStore> playerRef = getPlayerReferenceSomehow();
    Store<EntityStore> store = playerRef.getStore();
    store.tryRemoveComponent(playerRef, DeathComponent.getComponentType()); // Causes race conditions
    ```

## Data Pipeline
The flow of data for this command begins with user input and ends with a state change in the ECS, which in turn triggers game logic for respawning.

> Flow (Other Player Variant):
> Player Chat Input (`/respawn Steve`) -> Network Packet -> Server Command Parser -> CommandRegistry finds **PlayerRespawnCommand** -> Argument Parser populates `playerArg` -> `PlayerRespawnOtherCommand.executeSync` is invoked -> Task is scheduled via `world.execute()` -> **EntityStore** removes **DeathComponent** -> Game systems react to component removal -> Player respawns.

