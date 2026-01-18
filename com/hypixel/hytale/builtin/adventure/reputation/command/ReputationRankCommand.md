---
description: Architectural reference for ReputationRankCommand
---

# ReputationRankCommand

**Package:** com.hypixel.hytale.builtin.adventure.reputation.command
**Type:** Transient Handler

## Definition
```java
// Signature
public class ReputationRankCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The ReputationRankCommand class is a server-side command handler responsible for processing the `/reputation rank` command. It serves as the direct interface between a user (player or console) and the underlying reputation system for the purpose of querying a player's standing.

Architecturally, this class is a leaf node in the server's command tree. It extends AbstractTargetPlayerCommand, a specialized base class that abstracts away the complexity of parsing and resolving a target player from the command's arguments. This inheritance delegates the responsibility of player lookup to the framework, allowing ReputationRankCommand to focus solely on its core logic: retrieving and displaying reputation data.

Its primary collaborator is the ReputationPlugin, from which it fetches reputation values and ranks. This class does not contain any business logic for calculating reputation; it is strictly a presentation layer component that translates a user command into a query against the ReputationPlugin service and formats the result into a user-facing message.

## Lifecycle & Ownership
- **Creation:** An instance of ReputationRankCommand is created and registered by its parent command or plugin, typically during the server's plugin initialization phase. It is not intended for on-demand instantiation.
- **Scope:** The object's lifecycle is bound to the server's command registry. It persists for the entire server session, or until the parent plugin is disabled and its commands are unregistered.
- **Destruction:** The instance is marked for garbage collection when the server unregisters its commands, usually during a clean shutdown or plugin unload procedure.

## Internal State & Concurrency
- **State:** This class is effectively stateless with respect to command execution. It holds an immutable reference to its argument definition, the RequiredArg for the reputation group. All state required for execution, such as the target player and world context, is passed into the execute method by the command system.
- **Thread Safety:** The class itself contains no mutable state and is therefore thread-safe. Command execution is managed by the server's command processing system, which guarantees that the execute method is invoked in a thread-safe context, typically on the main server thread.

**Warning:** As execution likely occurs on the main server thread, the implementation of the execute method must be non-blocking to avoid impacting server performance.

## API Surface
The public contract is defined by its constructor for instantiation by the framework and the protected execute method, which serves as the callback for the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | protected void | O(Lookup) | Executes the command logic. Retrieves the reputation rank for the target player and specified group, sending a formatted message back to the command source. |

## Integration Patterns

### Standard Usage
This class is not designed to be used directly by developers. It is instantiated and managed exclusively by the server's command system as part of a plugin's command registration process. The standard interaction is through a user executing the command in-game or via the console.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new ReputationRankCommand()`. The command system is responsible for its lifecycle.
- **Direct Invocation:** Do not call the `execute` method directly. Doing so bypasses the command system's parsing, permission checks, and context injection, leading to unpredictable behavior and NullPointerExceptions.

## Data Pipeline
The flow of data for a typical command execution follows a clear, linear path from user input to system response.

> Flow:
> User Command String -> Server Command Parser -> **ReputationRankCommand** -> ReputationPlugin -> EntityStore Read -> **ReputationRankCommand** -> Message Formatter -> CommandContext -> User Chat/Console

