---
description: Architectural reference for WhereAmICommand
---

# WhereAmICommand

**Package:** com.hypixel.hytale.server.core.command.commands.player
**Type:** Transient

## Definition
```java
// Signature
public class WhereAmICommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The WhereAmICommand is a server-side command implementation responsible for reporting the location and state of a player entity. It serves as a concrete example of the engine's Command System, which uses an object-oriented approach where each command is a class instance rather than a simple function.

This class integrates directly with the server's core Entity Component System (ECS) and World management. Its primary function is to query entity components—specifically TransformComponent and HeadRotation—to gather positional and rotational data. It then formats this data into a user-friendly Message object and sends it back to the command's originator.

A key architectural feature demonstrated here is the use of a nested inner class, WhereAmIOtherCommand, to represent a "usage variant". This allows the base command `/whereami` to be extended with `/whereami <player>` functionality without creating a separate, top-level command. This pattern promotes code reuse and logical grouping of related command actions.

## Lifecycle & Ownership
-   **Creation:** An instance of WhereAmICommand is created once during server initialization by the central Command System registry. The constructor is responsible for self-registration of its name, description, required permissions, and its subcommand variants.
-   **Scope:** The command object is a singleton managed by the Command System. It persists for the entire lifetime of the server session.
-   **Destruction:** The object is dereferenced and garbage collected only when the server shuts down and the Command System is dismantled.

## Internal State & Concurrency
-   **State:** This class is **stateless**. It contains no mutable instance fields. All data required for an operation is provided through the arguments of the `execute` method, primarily via the CommandContext. Its fields are final, immutable Message constants.

-   **Thread Safety:** The WhereAmICommand instance itself is thread-safe due to its stateless nature. However, its methods perform read operations on the game world, which is not thread-safe.

    **WARNING:** Direct access to an entity's components or the World object must be synchronized with the main server tick loop. The nested WhereAmIOtherCommand correctly demonstrates this pattern by scheduling its logic onto the target world's execution queue via `world.execute()`. Failure to follow this pattern will result in data races, inconsistent state reads, and server instability. The base `execute` method, inherited from AbstractPlayerCommand, is implicitly called on the correct world thread by the command system.

## API Surface
The public contract is defined by its constructor for registration and the `execute` method for invocation by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WhereAmICommand() | constructor | O(1) | Initializes the command, setting its name, description, and permission requirements. Registers the `WhereAmIOtherCommand` variant. |
| execute(...) | protected void | O(1) | Framework-invoked method. Gathers positional data for the command sender and formats the response. |

## Integration Patterns

### Standard Usage
This command is not intended to be used directly from code. It is invoked by the server's command dispatcher when a player types the command into chat. The system handles parsing, permission checks, and invocation.

A conceptual view of the system's interaction:
```java
// This is a conceptual representation of the Command Dispatcher's role.
// Do NOT invoke commands this way in application code.

// 1. Player input is parsed.
String commandName = "whereami";
CommandContext context = createPlayerContext(player);

// 2. The dispatcher finds the registered command object.
CommandBase command = commandRegistry.find(commandName);

// 3. The dispatcher invokes the command.
if (command.hasPermission(context.getSender())) {
    command.execute(context);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new WhereAmICommand()` in game logic. The command system handles the lifecycle of all command objects. Manual instantiation will result in a non-functional command that is not registered with the server.
-   **Manual Invocation:** Avoid calling the `execute` method directly. This bypasses the command system's essential preprocessing, including argument parsing and permission validation, which can lead to unpredictable behavior and security vulnerabilities.
-   **Cross-Thread World Access:** As highlighted in the concurrency section, never access entity components or world state from an external thread without scheduling the operation via `world.execute()`. The nested class provides the canonical example of the correct approach.

## Data Pipeline
The command acts as a request-response handler within the server, transforming a player's text input into a structured data report.

> Flow:
> Client Chat Input (`/whereami`) -> Server Network Packet -> Command Dispatcher -> **WhereAmICommand.execute** -> Read from ECS Store (TransformComponent, etc.) -> Format `Message` object -> CommandContext.sendMessage -> Server Network Packet -> Client Chat Display

