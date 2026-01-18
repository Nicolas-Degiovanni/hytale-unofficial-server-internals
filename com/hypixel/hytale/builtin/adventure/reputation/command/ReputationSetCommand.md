---
description: Architectural reference for ReputationSetCommand
---

# ReputationSetCommand

**Package:** com.hypixel.hytale.builtin.adventure.reputation.command
**Type:** Transient

## Definition
```java
// Signature
public class ReputationSetCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The ReputationSetCommand class provides a server-side, administrative command-line interface for directly manipulating a player's reputation value. It serves as a bridge between the server's command system and the underlying reputation game logic managed by the ReputationPlugin.

Architecturally, this class is a *Terminal Adapter*. It translates a text-based command, issued by a privileged user, into a direct, programmatic call to the core reputation system. It inherits from AbstractTargetPlayerCommand, which abstracts away the boilerplate logic of parsing a target player from the command's arguments.

Its primary responsibilities are:
1.  Defining the command's syntax, including its name ("set") and required arguments (reputation group and integer value).
2.  Parsing and validating these arguments from a given CommandContext during execution.
3.  Retrieving the target player's current reputation value.
4.  Calculating the necessary delta to reach the desired value.
5.  Delegating the final state-changing operation to the ReputationPlugin singleton.
6.  Providing localized feedback to the user who issued the command.

This class does not contain any reputation logic itself; it is purely an entry point for administrative action.

## Lifecycle & Ownership
-   **Creation:** A single instance of ReputationSetCommand is instantiated by the server's command registration system when the parent ReputationPlugin is loaded. It is then registered as a subcommand within the primary `/reputation` command tree.
-   **Scope:** The object instance persists for the entire server session, for as long as the command is registered. It is functionally stateless between executions.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the server shuts down or the ReputationPlugin is unloaded, which unregisters all its associated commands.

## Internal State & Concurrency
-   **State:** This class is effectively **immutable** after construction. The fields `reputationGroupIdArg` and `valueArg` are final and define the command's argument structure. They do not hold state related to any specific execution. All state required for an operation is passed into the `execute` method via its parameters.
-   **Thread Safety:** This class is not thread-safe and is not designed for concurrent access. Command execution is expected to occur exclusively on the main server thread as part of the game loop's command processing phase. Any attempt to invoke its methods from an asynchronous task will lead to race conditions and world state corruption.

## API Surface
The public contract is limited to the constructor and the overridden `execute` method, which is invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | void | O(1) | Executes the command. This method is the sole entry point for the command's logic and must only be called by the server's command handler. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is designed to be invoked via the server console or by an in-game user with sufficient permissions.

The standard usage is via the command line:

`/reputation set <PlayerName> <ReputationGroupID> <Value>`

Example:
`/reputation set Steve hytale:example_faction 500`

For programmatic modification of player reputation, developers **must** interact with the ReputationPlugin directly.

```java
// CORRECT: Programmatic modification via the plugin
ReputationPlugin plugin = ReputationPlugin.get();
int current = plugin.getReputationValue(store, playerRef, reputationGroup.getId());
int desired = 500;
plugin.changeReputation(player, reputationGroup.getId(), desired - current, store);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ReputationSetCommand()`. The command system manages the lifecycle of command objects. Creating an instance has no effect unless it is properly registered with the command tree.
-   **Direct Invocation:** Never call the `execute` method directly. Doing so bypasses critical infrastructure, including argument parsing, permission validation, and context setup, which will result in NullPointerExceptions and invalid server state.

## Data Pipeline
The flow of data for this command begins with user input and terminates with a change in the world state and user feedback.

> Flow:
> Command String -> Server Command Parser -> **ReputationSetCommand.execute()** -> ReputationPlugin -> EntityStore Update -> CommandContext Feedback Message

