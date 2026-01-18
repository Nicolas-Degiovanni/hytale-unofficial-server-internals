---
description: Architectural reference for NPCGiveCommand
---

# NPCGiveCommand

**Package:** com.hypixel.hytale.server.npc.commands
**Type:** Transient

## Definition
```java
// Signature
public class NPCGiveCommand extends NPCWorldCommandBase {
```

## Architecture & Concepts
The NPCGiveCommand class is a concrete implementation of the Command Pattern, designed to integrate with the server's core command processing system. It provides game administrators and developers with a console or chat-based mechanism to equip an NPCEntity with a specified item.

Architecturally, this class serves as a thin orchestration layer. It does not contain complex game logic itself. Instead, its primary responsibilities are:
1.  **Command Definition:** It declares the command name ("give"), its description, and defines the required arguments—in this case, an item asset.
2.  **Argument Parsing:** It leverages the framework's argument system (ArgTypes.ITEM_ASSET) to parse a string identifier into a concrete Item asset reference.
3.  **Contextual Execution:** By extending NPCWorldCommandBase, it inherits the necessary boilerplate to ensure the command is executed within a valid context, targeting a specific NPCEntity within a World.
4.  **Logic Delegation:** The core logic of modifying the NPCEntity's state is delegated to the RoleUtils utility class. This maintains a strong separation of concerns, keeping command classes focused on input processing and leaving entity manipulation to dedicated systems.

The class also includes a nested subcommand, GiveNothingCommand, which demonstrates a compositional approach to building complex command hierarchies. This allows for more expressive and organized command structures, such as `/npc give nothing`.

### Lifecycle & Ownership
-   **Creation:** An instance of NPCGiveCommand is created once during server initialization by the central CommandManager or a similar registration authority. It is discovered and registered as an available server command.
-   **Scope:** The command object is long-lived, persisting in the server's command registry for the entire server session. It is designed to be stateless so a single instance can handle all invocations.
-   **Destruction:** The object is dereferenced and becomes eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is effectively stateless and immutable after construction. The field itemArg is an argument definition object, which is configured in the constructor and not modified thereafter. The execute method operates exclusively on parameters passed into it by the command system, ensuring no state is carried between invocations.

-   **Thread Safety:** The object itself is thread-safe due to its stateless nature. However, the execute method performs mutations on shared game state (NPCEntity, World). The command system framework **MUST** ensure that all command execution is serialized and occurs on the main server thread. Calling execute from an asynchronous task or a different thread will lead to severe concurrency violations, data corruption, and server instability.

## API Surface
The primary contract is the protected execute method, which is invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NPCGiveCommand() | Constructor | O(1) | Initializes the command, its subcommand, and defines the required item argument. |
| execute(...) | protected void | O(1) | Parses the item argument and delegates to RoleUtils to equip it on the target NPC. |
| GiveNothingCommand | public static class | N/A | A nested subcommand to clear the item from an NPC's hand. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. It is automatically registered and invoked by the server's command handling system in response to user input.

A server administrator or player with sufficient permissions would use this command via a chat or console interface.

**Example User Invocation:**
```
/npc select <npc_id>
/npc give hytale:iron_sword
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new NPCGiveCommand()` in game logic. The command system is responsible for its lifecycle. Creating instances manually will have no effect as they will not be registered.
-   **Manual Execution:** Never call the `execute` method directly. This bypasses critical framework functionality, including permission checks, argument parsing, context validation, and thread safety guarantees provided by the command dispatcher.

## Data Pipeline
The flow for this command begins with user input and ends with a change in the world state. The NPCGiveCommand acts as a key component in the middle of this pipeline.

> Flow:
> User Input (`/npc give hytale:iron_sword`) -> Server Network Layer -> Command Dispatcher -> **NPCGiveCommand** -> Argument Parser (resolves `hytale:iron_sword` to an Item object) -> `execute` method -> RoleUtils.setItemInHand -> NPCEntity Component Update -> World State Change

