---
description: Architectural reference for PlayerStatsDumpCommand
---

# PlayerStatsDumpCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.stats
**Type:** Transient

## Definition
```java
// Signature
public class PlayerStatsDumpCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The PlayerStatsDumpCommand is a server-side command handler responsible for outputting the raw component data of a specific player entity. Architecturally, this class functions as a specialized **Adapter** within the server's command processing system. It translates a user-facing command, intended to operate on a player, into a call to a more generic, underlying entity-dumping mechanism.

Its direct parent, AbstractTargetPlayerCommand, abstracts away the boilerplate logic of parsing and resolving a player name from the command's arguments. This allows PlayerStatsDumpCommand to focus exclusively on its core task: orchestrating the data dump for the resolved player entity. It does not contain any data dumping logic itself; it delegates this responsibility entirely to the EntityStatsDumpCommand utility, reinforcing the principle of separation of concerns.

## Lifecycle & Ownership
- **Creation:** A single instance of PlayerStatsDumpCommand is created by the server's command registration system during the server bootstrap sequence. The system typically discovers command classes via reflection or a predefined list.
- **Scope:** The object is a stateless singleton managed by the CommandRegistry. It persists for the entire lifetime of the server session.
- **Destruction:** The instance is eligible for garbage collection only when the server is shutting down and the central CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no member fields and its behavior is determined entirely by the arguments passed to its execute method.
- **Thread Safety:** The class instance itself is inherently thread-safe due to its immutability. However, the execute method is **not safe to call from arbitrary threads**. It reads from and interacts with live game state objects like World and EntityStore. The server's command dispatcher **must** ensure that all command execution is serialized and performed on the main server thread to prevent critical race conditions and data corruption.

## API Surface
The primary contract is the `execute` method, which is invoked by the command system framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, sourceRef, ref, playerRef, world, store) | void | O(N) | Executes the command logic. Delegates to the generic EntityStatsDumpCommand to serialize and output all component data for the target player. Complexity is proportional to N, the number of stat components on the entity. |

## Integration Patterns

### Standard Usage
This class is not designed to be used directly in code. It is invoked automatically by the server's command dispatcher in response to a user typing the command into the console or chat.

The conceptual flow within the dispatcher resembles the following:

```java
// PSEUDO-CODE: Represents the server's command dispatch loop

// User types: /player stats dump Notch
String userInput = "/player stats dump Notch";

// System parses and finds the correct command handler
AbstractCommand handler = commandRegistry.find("player stats dump"); // Returns PlayerStatsDumpCommand instance

// System resolves arguments and invokes the handler
CommandContext context = create_context_for_user(userInput);
handler.process(context); // This eventually calls the execute method with resolved arguments
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PlayerStatsDumpCommand()`. The instance has no value outside the command registration system. It is a stateless handler, not a general-purpose utility.
- **Manual Invocation:** Never call the `execute` method directly. The parameters it requires, such as CommandContext, PlayerRef, and World, are deeply integrated with the server's state and are non-trivial to construct correctly. Bypassing the command dispatcher can lead to state corruption, null pointer exceptions, and security bypasses.

## Data Pipeline
The flow of data for this command begins with user input and ends with a message sent back to that user. The PlayerStatsDumpCommand acts as a routing step in the middle of this process.

> Flow:
> User Command String -> Command Dispatcher -> Argument Parser (resolves player) -> **PlayerStatsDumpCommand.execute** -> EntityStatsDumpCommand.dumpEntityStatsData -> EntityStore (data retrieval) -> Formatted String -> CommandContext (sends reply) -> User Chat/Console

