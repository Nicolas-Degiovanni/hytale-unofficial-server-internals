---
description: Architectural reference for BrushConfigDebugStepCommand
---

# BrushConfigDebugStepCommand

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.commands
**Type:** Transient

## Definition
```java
// Signature
public class BrushConfigDebugStepCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The **BrushConfigDebugStepCommand** is an implementation of the Command Pattern, providing a user-facing entry point to manipulate the state of a server-side process. It functions as a debugging tool, allowing a player to manually advance the execution of a scripted brush operation sequence one step at a time.

Architecturally, this class is a thin, stateless controller. Its sole responsibility is to translate a player's chat command into a method call on a stateful service object, the **BrushConfigCommandExecutor**. It contains no core logic for brush execution itself; it is purely a delegation and feedback mechanism. This separation of concerns ensures that the command handler remains simple and decoupled from the complex state machine of the brush execution system.

This command is a critical component of the builder tools debugging workflow, enabling developers to inspect the step-by-step application of complex, multi-stage brush operations on the world.

## Lifecycle & Ownership
- **Creation:** A single instance of **BrushConfigDebugStepCommand** is instantiated by the server's CommandSystem during the initial bootstrap phase. The system discovers and registers all classes that extend **AbstractPlayerCommand**.
- **Scope:** The command object itself is a singleton that persists for the entire server session, managed by the CommandSystem. The execution context, however, is transactional; the *execute* method is invoked once per command usage and holds no state between calls.
- **Destruction:** The singleton instance is discarded when the server shuts down and the CommandSystem is de-allocated.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its only member field, *numStepsArg*, is an immutable definition object used for argument parsing. All state that this command reads and manipulates is external, located within the player-specific **PrototypePlayerBuilderToolSettings** and its associated **BrushConfigCommandExecutor**.

- **Thread Safety:** This command is **not thread-safe** and is designed to be executed exclusively on the server's main game thread. All interactions with the world, entity stores, and player settings assume single-threaded access, which is the prevailing concurrency model for Hytale's server logic. Invoking its methods from any other thread will lead to race conditions and world state corruption.

## API Surface
The public contract is defined by its role as a command handler. Direct invocation outside the CommandSystem is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(N) | Executes the command. N is the number of steps requested. Fetches the player's brush executor and invokes its *step* method N times. Sends feedback messages to the player. |

## Integration Patterns

### Standard Usage
A user with appropriate permissions invokes this command through the in-game chat console to debug a scripted brush that has been previously started in debug mode.

```
// Player-side action (in-game chat)
/step 5
```
This command triggers the server-side execution flow, advancing the active brush by five operations and reporting the results back to the player.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BrushConfigDebugStepCommand()`. The command system manages the lifecycle of command objects. A manually created instance will not be registered and cannot be invoked.
- **Manual Execution:** Do not call the *execute* method directly. This method relies on a fully populated **CommandContext** object, which is only provided by the server's command dispatching framework. Manual invocation will bypass argument parsing and context injection, resulting in runtime exceptions.
- **State Assumption:** This command is part of a multi-step workflow. Invoking it when a brush is not in an active debug session will have no effect other than sending an error message to the player. It must be preceded by an action that initializes the **BrushConfigCommandExecutor**.

## Data Pipeline
The command acts as a controller in a user-initiated data flow. It receives a command, triggers a state change in a separate system, and then formats the result of that change into a message for the user.

> Flow:
> Player Chat Input (`/step 5`) → Server Command Parser → **BrushConfigDebugStepCommand.execute()** → PrototypePlayerBuilderToolSettings → BrushConfigCommandExecutor.step() → World State Mutation → Message Generation → PlayerRef.sendMessage() → Client UI Update

