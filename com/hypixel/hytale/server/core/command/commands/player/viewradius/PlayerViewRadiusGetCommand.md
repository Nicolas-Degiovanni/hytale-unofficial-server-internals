---
description: Architectural reference for PlayerViewRadiusGetCommand
---

# PlayerViewRadiusGetCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.viewradius
**Type:** Transient

## Definition
```java
// Signature
public class PlayerViewRadiusGetCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The PlayerViewRadiusGetCommand is a server-side command handler that operates within the Hytale Command System. It embodies the Command design pattern, encapsulating a specific, read-only action: retrieving a player's view radius settings.

As a subclass of AbstractTargetPlayerCommand, its design is specialized for operations that target a specific player entity within the game world. It does not modify any state; its sole responsibility is to query data from the Entity Component System (ECS) and report it back to the command's issuer.

This class acts as a data retrieval bridge between the command system and the ECS. It accesses the central EntityStore to fetch the Player and EntityViewer components associated with the target player, extracts the relevant view distance values, and formats them into a user-facing message.

### Lifecycle & Ownership
-   **Creation:** A single instance of PlayerViewRadiusGetCommand is instantiated by the server's command registration system during the server bootstrap sequence. It is registered under the command path `player viewradius get`.
-   **Scope:** The object instance persists for the entire lifetime of the server. It is a stateless handler designed to be invoked repeatedly.
-   **Destruction:** The instance is eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is **stateless**. It contains no mutable instance fields. All necessary data, such as the target player and the world state, is provided via the arguments of the `execute` method. This stateless design is crucial for reusability and safety within the command system.
-   **Thread Safety:** The class is inherently thread-safe. However, the `execute` method is invoked by the command system, which must ensure that all operations on the provided Store and World objects are synchronized with the main server thread. Direct, asynchronous invocation of `execute` from an unmanaged thread would lead to severe concurrency violations and data corruption within the ECS.

## API Surface
The public contract is fulfilled by inheriting from the command system's base classes. The core logic is contained within the protected `execute` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, sourceRef, ref, playerRef, world, store) | void | O(1) | Retrieves view radius data from the target player's components and sends a formatted message to the command source. Asserts that required components exist. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly. It is invoked by the server's command dispatcher when a user or system executes the corresponding command string. The framework provides the context and entity references.

A user would trigger this command via the in-game console:
```sh
# Example command entered by a player or console
/player SomePlayerName viewradius get
```

The system then resolves this command and invokes the handler:
```java
// Conceptual representation of the command system invoking the handler
CommandContext context = createCommandContextFor("/player SomePlayerName viewradius get");
PlayerViewRadiusGetCommand handler = commandRegistry.getHandlerFor(context);

// The framework finds the target player and provides all necessary arguments
handler.execute(context, /* source */, /* target */, /* ... */);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PlayerViewRadiusGetCommand()`. The command system manages the lifecycle of handler instances. Manual creation serves no purpose as it will not be registered to handle commands.
-   **Manual Invocation:** Do not call the `execute` method directly. Doing so bypasses critical framework functionality, including permission checks, argument parsing, target entity resolution, and thread synchronization. This will result in unpredictable behavior and likely throw a NullPointerException or IllegalStateException.

## Data Pipeline
The command acts as a specific step in a larger data flow, transforming a user command into a data-driven response.

> Flow:
> User Command String -> Command Parser -> **PlayerViewRadiusGetCommand** -> EntityStore Component Query -> Message Builder -> CommandContext -> Network Layer -> Client Chat UI

