---
description: Architectural reference for ReputationValueCommand
---

# ReputationValueCommand

**Package:** com.hypixel.hytale.builtin.adventure.reputation.command
**Type:** Transient

## Definition
```java
// Signature
public class ReputationValueCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The ReputationValueCommand class implements a server-side administrative command. It serves as a direct interface for server operators or developers to query the reputation value of a specific player for a given reputation group.

Architecturally, this class is a leaf node in the server's command processing system. It extends AbstractTargetPlayerCommand, a specialized base class that abstracts away the boilerplate logic of parsing a target player from the command's arguments. Its sole responsibility is to parse its own specific arguments, retrieve the relevant data from the core reputation system, and format a response message.

It does not contain any game logic itself. Instead, it acts as a bridge, translating a text-based command into a programmatic query against the ReputationPlugin. This decouples the user-facing command interface from the underlying game mechanics, allowing the reputation system to evolve without breaking the command's contract.

A key component is its use of the AssetArgumentType. This argument parser is responsible for resolving a string identifier provided by the user (the reputation group's ID) into a fully loaded ReputationGroup asset object, ensuring type safety and asset validity before the core logic is ever executed.

### Lifecycle & Ownership
- **Creation:** An instance of ReputationValueCommand is created by the server's command registration system when the ReputationPlugin is loaded. The system scans for command classes and instantiates them to build a central command registry.
- **Scope:** The object instance persists for the entire server session, held within the command registry. While the object itself is long-lived, its role is that of a transient handler; no state is maintained between command executions.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the server shuts down or the ReputationPlugin is unloaded, at which point the command is removed from the registry.

## Internal State & Concurrency
- **State:** This class is effectively stateless regarding command execution. Its fields, such as reputationGroupIdArg, are configuration details defined during construction and are immutable thereafter. They define the command's syntax and argument requirements but do not change during runtime. All state required for execution (the target player, the world, the command context) is passed as parameters to the execute method.
- **Thread Safety:** This class is not thread-safe and is not designed to be. The execute method interacts directly with core game state components like World and EntityStore, which must only be accessed from the main server thread. The command system framework guarantees that all command executions are synchronized with the server's main game loop, preventing concurrency issues.

**WARNING:** Calling the execute method from any thread other than the main server thread will lead to race conditions, data corruption, and server instability.

## API Surface
The primary contract is the overridden execute method, which is invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, sourceRef, ref, playerRef, world, store) | void | O(1) | Executes the command logic. Parses the ReputationGroup argument, queries the ReputationPlugin for the value, and sends a formatted response to the command source. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is a framework component, automatically registered and executed by the server's command handler. A user with appropriate permissions would trigger it by typing the command into the game's chat console.

Example command entered by a user:

`/reputation value PlayerName hytale:faction_a`

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ReputationValueCommand()` in your own code. The resulting object will not be registered with the command system and will be non-functional.
- **Manual Execution:** Do not call the `execute` method directly. This bypasses critical framework features like argument parsing, permission checks, and thread safety guarantees. All interaction should be through the server's command dispatch system.

## Data Pipeline
The flow of data for a successful command execution follows a clear path from user input to user feedback.

> Flow:
> User Chat Input -> Server Command Parser -> **ReputationValueCommand.execute()** -> ReputationPlugin.getReputationValue() -> EntityStore Lookup -> Formatted Message -> CommandContext.sendMessage() -> Client Chat UI

