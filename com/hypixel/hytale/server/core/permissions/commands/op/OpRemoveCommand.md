---
description: Architectural reference for OpRemoveCommand
---

# OpRemoveCommand

**Package:** com.hypixel.hytale.server.core.permissions.commands.op
**Type:** Transient

## Definition
```java
// Signature
public class OpRemoveCommand extends CommandBase {
```

## Architecture & Concepts
The OpRemoveCommand class is a concrete implementation of the Command Pattern, designed to integrate into the server's command processing system. It is not a persistent service but rather a stateless "verb" that encapsulates the specific logic for revoking operator (OP) permissions from a player.

Architecturally, this command acts as a transactional controller that orchestrates changes between high-level server systems. It receives its operational context via the CommandContext parameter during execution and interacts directly with two critical, server-wide singletons:
- **PermissionsModule:** The authoritative source for all player permissions. OpRemoveCommand issues a write operation to this module to mutate a player's group membership.
- **Universe:** The primary service for querying the state of all online entities, including players. OpRemoveCommand uses this for a read operation to find the target player and deliver a direct notification.

The class itself is stateless. All information required for its execution is either provided at runtime or retrieved from the aforementioned global systems.

### Lifecycle & Ownership
- **Creation:** A single instance of OpRemoveCommand is instantiated by the server's central CommandRegistry during the bootstrap phase. The registry discovers all CommandBase subclasses and registers them for later invocation.
- **Scope:** The definition object persists for the entire server session, held as a reference within the CommandRegistry. The execution itself is ephemeral; a new CommandContext is created for each command invocation and discarded immediately after.
- **Destruction:** The object is dereferenced and garbage collected only when the server shuts down and the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** The class is effectively immutable after its initial construction. Its fields, such as the RequiredArg definition, are configured in the constructor and are not modified during the command's lifecycle. It holds no per-execution state; all runtime data is managed within the CommandContext.
- **Thread Safety:** **This class is not thread-safe and must not be treated as such.** The method name executeSync explicitly signals that its logic must be run on the main server thread. It performs unsafe, non-atomic operations on global singletons (PermissionsModule, Universe) which are not designed for concurrent access. The server's command execution framework is responsible for ensuring this contract is upheld by queuing and dispatching all command executions serially on the main game loop.

## API Surface
The primary API is the contract inherited from CommandBase, which is internal to the command system. Direct invocation is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(log N) | Executes the permission revocation logic. Complexity is driven by lookups in the PermissionsModule and Universe. Mutates global permission state. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by developers. It is invoked automatically by the server's command handler when a user with sufficient permissions executes the corresponding command in-game or via the console.

The following example illustrates the *conceptual* flow of how the framework dispatches the command, not how a developer would typically use it.

```java
// This code is representative of the server's internal command dispatcher.
// Do not replicate this pattern in game logic or plugins.
CommandSystem commandSystem = server.getService(CommandSystem.class);
CommandSource source = getConsoleCommandSource();

// The dispatcher parses the string, finds the OpRemoveCommand,
// resolves arguments, checks permissions, and finally calls executeSync.
commandSystem.dispatch(source, "op remove SomePlayer");
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new OpRemoveCommand()`. The command system manages the lifecycle of command objects. Manual instantiation creates a disconnected object that is not registered to handle any input.
- **Manual Execution:** Never call the executeSync method directly. Doing so bypasses the entire command framework, including critical pre-flight checks for permissions, argument validation, and standardized exception handling. This can lead to a corrupted server state.
- **Asynchronous Access:** Do not invoke this command from a separate thread. It interacts with non-thread-safe systems and will cause race conditions and server instability.

## Data Pipeline
The execution of this command is the final step in a data flow that begins with user input and results in a change to the server's core permission state.

> Flow:
> Player Chat Input -> Network Packet -> Server Command Parser -> **OpRemoveCommand.executeSync()** -> PermissionsModule (State Mutation) -> Universe (Player Lookup) -> Network Packet (Feedback to clients)

