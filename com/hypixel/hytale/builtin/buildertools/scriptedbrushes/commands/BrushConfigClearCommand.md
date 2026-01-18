---
description: Architectural reference for BrushConfigClearCommand
---

# BrushConfigClearCommand

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.commands
**Type:** Command Handler

## Definition
```java
// Signature
public class BrushConfigClearCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The BrushConfigClearCommand class is a concrete implementation of the Command Pattern, designed to integrate directly into the server's command processing system. It serves as a user-facing entry point for players to manage the state of their scripted builder tools.

Its primary architectural role is to act as a transactional controller that modifies a player's specific **PrototypePlayerBuilderToolSettings**. When a player executes the associated command, this class is responsible for safely clearing all configured brush operations and disabling the scripted brush feature for that player. It is not a persistent service but a stateless handler that is invoked on-demand by the server's command dispatcher.

A critical design feature is the pre-execution state check. The command verifies that the player's brush is not currently active before applying any changes, preventing data corruption and undefined behavior during an in-progress tool operation.

## Lifecycle & Ownership
- **Creation:** A single instance of BrushConfigClearCommand is instantiated by the server's core command registry during the server bootstrap sequence. It is discovered and registered automatically alongside all other built-in commands.
- **Scope:** The object exists as a singleton for the entire server session. The same instance is reused for every execution of the *clear* or *disable* brush command, regardless of which player invokes it.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the command registry is dismantled.

## Internal State & Concurrency
- **State:** The BrushConfigClearCommand instance itself is effectively immutable. Its configuration, such as its name and aliases, is defined in the constructor and does not change during its lifecycle. However, its purpose is to mutate the state of external, per-player objects, namely the **PrototypePlayerBuilderToolSettings**.

- **Thread Safety:** This class is designed to be executed within the server's single-threaded, per-player command execution model. The `execute` method is not inherently thread-safe and must not be invoked from arbitrary threads. Concurrency control is managed cooperatively through a state check (`isCurrentlyExecuting()`) on the target settings object. This check functions as a non-blocking lock, ensuring that the command does not interfere with an ongoing `ToolOperation`.

## API Surface
The public contract is defined by its role as a command handler. Direct invocation is strongly discouraged.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(N) | Executes the command logic. Complexity is O(N) where N is the number of operations in the player's brush configuration, due to the `clear()` calls on the operation lists. Throws exceptions if the context is invalid. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. The server's command system invokes it automatically in response to player input.

> **System Invocation Flow**
> 1. A player types `/brush clear` or `/brush disable` in chat.
> 2. The server's command parser identifies `BrushConfigClearCommand` as the registered handler.
> 3. The system populates a `CommandContext` with the relevant player and world data.
> 4. The system invokes the `execute` method on the singleton instance of this class.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BrushConfigClearCommand()`. The command system manages the lifecycle of all command handlers. Manual instantiation will result in an object that is not registered and will never be called.
- **Direct Invocation:** Do not call the `execute` method from other parts of the codebase. Doing so bypasses critical infrastructure, including permission checks, context validation, and the server's threading model, which can lead to severe state corruption.
- **State Modification:** Do not attempt to modify the state of this command object after its construction. It is designed to be a stateless handler.

## Data Pipeline
This command initiates a state change that flows from player input to a modification of a player's data, with feedback sent back to the client.

> Flow:
> Player Chat Input -> Command Parser -> **BrushConfigClearCommand.execute()** -> ToolOperation (Settings Lookup) -> PrototypePlayerBuilderToolSettings (State Cleared) -> PlayerRef (Send Confirmation Message) -> Client UI

