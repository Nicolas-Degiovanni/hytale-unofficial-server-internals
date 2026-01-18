---
description: Architectural reference for GameModeCommand
---

# GameModeCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player
**Type:** Transient

## Definition
```java
// Signature
public class GameModeCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The GameModeCommand class is a server-side command handler responsible for processing the `/gamemode` and `/gm` commands. It serves as a direct interface between player input and the server's core game state, specifically the game mode property of a Player entity.

This class is a concrete implementation within the server's Command System. It extends AbstractPlayerCommand, a specialized base class for commands that are executed by and primarily target a player entity. This inheritance provides the execution context, including a direct reference to the executing player's entity, without requiring manual lookups.

A key architectural pattern demonstrated here is the use of **Usage Variants**. The primary class handles the self-targeting variant (e.g., `/gamemode creative`). A private nested class, GameModeOtherCommand, is registered as a usage variant to handle the other-targeting variant (e.g., `/gamemode creative Notch`). This compositional approach keeps the logic for different command signatures cleanly separated within the same file, avoiding complex conditional branching in a single execute method.

Interaction with the game world is mediated entirely through the Entity Component System (ECS). The command does not hold a direct reference to a Player object. Instead, it receives a Store and a Ref to an EntityStore, from which it retrieves the Player component. All state modifications, such as setting the new game mode, are performed through the ECS API (Player.setGameMode), ensuring data integrity and proper event propagation.

## Lifecycle & Ownership
-   **Creation:** A single instance of GameModeCommand is instantiated by the server's CommandRegistry during the server bootstrap phase. The system scans for classes annotated or designated as commands and creates long-lived instances.
-   **Scope:** The GameModeCommand object is a stateless singleton that persists for the entire server session. It is held in memory by the CommandRegistry. The *execution* of the command is transient; its methods are invoked on-demand for each matching player command input.
-   **Destruction:** The instance is eligible for garbage collection only when the server is shutting down and the CommandRegistry is cleared.

## Internal State & Concurrency
-   **State:** The GameModeCommand instance is effectively immutable and stateless. Its fields, such as RequiredArg definitions, are final and initialized in the constructor. All state required for execution (e.g., the target player, the desired game mode) is provided at runtime via the CommandContext.

-   **Thread Safety:** The class instance is inherently thread-safe due to its stateless nature. However, the execution logic operates on mutable game state and must adhere to strict threading models.
    -   The base command's `execute` method, inherited from AbstractPlayerCommand, is guaranteed by the framework to be called on the correct World thread for the *executing* player.
    -   The `GameModeOtherCommand` variant demonstrates a critical concurrency pattern: when targeting *another* player, who may reside in a different world or thread, it explicitly retrieves the target's World and schedules the state modification using `world.execute`. This prevents cross-thread modification of ECS components and ensures all game logic remains thread-safe.

    **WARNING:** Bypassing the `world.execute` scheduler when modifying entities in other worlds will lead to severe concurrency issues, including data corruption and server crashes.

## API Surface
The primary contract is fulfilled by the `execute` method, which is invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Executes the logic to change the command-issuer's game mode. Interacts with the ECS. |
| GameModeOtherCommand.executeSync(context) | void | O(1) | Executes logic to change another player's game mode, dispatching the work to the target's world thread. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by developers. It is automatically discovered and invoked by the server's command dispatcher in response to player chat input. The system handles parsing, permission checks, and context creation.

A developer would interact with this system by creating a new command class following the same pattern:
```java
// Conceptual example of creating a similar command
public class HealCommand extends AbstractPlayerCommand {

    public HealCommand() {
        super("heal", "Heals the player.");
        requirePermission(HytalePermissions.fromCommand("heal.self"));
    }

    @Override
    protected void execute(@Nonnull CommandContext context, @Nonnull Store<EntityStore> store, @Nonnull Ref<EntityStore> ref, @Nonnull PlayerRef playerRef, @Nonnull World world) {
        Health healthComponent = store.getComponent(ref, Health.getComponentType());
        if (healthComponent != null) {
            // Logic to modify the component
            context.sendMessage(Message.translation("server.commands.heal.success"));
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new GameModeCommand()`. The command system manages the lifecycle of command instances. Manually creating one will result in a non-functional object that is not registered to handle any chat input.
-   **Manual Execution:** Do not call the `execute` method directly. This bypasses critical infrastructure, including permission validation, argument parsing, and thread safety guarantees provided by the command dispatcher.
-   **Cross-Thread State Modification:** As seen in GameModeOtherCommand, never modify an entity's components directly from the command dispatcher thread. Always schedule the modification on the entity's owning World thread.

## Data Pipeline
The flow of data from player input to game state change is managed by the server's command processing pipeline.

> Flow:
> Player Chat Input (`/gm creative`) -> Network Layer -> Command Dispatcher -> **GameModeCommand** -> Argument Parser (`creative` -> GameMode.CREATIVE) -> World Thread Scheduler -> ECS State Update (Player.setGameMode) -> State Synchronization Packet -> Client Visual Update

