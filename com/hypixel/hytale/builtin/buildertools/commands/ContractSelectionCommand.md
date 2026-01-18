---
description: Architectural reference for ContractSelectionCommand
---

# ContractSelectionCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class ContractSelectionCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The ContractSelectionCommand class is a command adapter within the server's command processing system. It serves as a user-facing entry point for a specific function of the Builder Tools plugin: shrinking a player's selection volume.

Architecturally, this class does not contain the core logic for world modification. Instead, it acts as a translator and a delegator. Its primary responsibilities are:
1.  **Argument Parsing:** Defining and parsing the required *distance* and optional *axis* arguments from the player's command input.
2.  **Context Validation:** Verifying that the player is in a valid state to perform a builder command by checking against PrototypePlayerBuilderToolSettings.
3.  **Intent Calculation:** Determining the direction of the contraction. If an axis is not specified, it dynamically uses the direction the player is looking, retrieved from the HeadRotation component.
4.  **Deferred Execution:** Enqueuing the final operation into the BuilderToolsPlugin's work queue. This is a critical design pattern that decouples the command input from the potentially expensive world modification logic, preventing the command system from blocking on heavy computations.

This class is a thin layer that connects the generic server command system to the specialized Builder Tools subsystem.

### Lifecycle & Ownership
-   **Creation:** A single instance of ContractSelectionCommand is instantiated by the server's command registry during plugin loading. The command system discovers and registers this object based on its class definition.
-   **Scope:** The registered instance persists for the entire server session, or until the BuilderToolsPlugin is unloaded. It is not created per-command-execution.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the server shuts down or the parent plugin is disabled, at which point it is removed from the command registry.

## Internal State & Concurrency
-   **State:** This class is effectively stateless in the context of a command execution. The fields distanceArg and axisArg are final and define the command's argument structure; they are configured once in the constructor and are immutable thereafter. All state required for execution (player, world, arguments) is passed into the execute method via the CommandContext.
-   **Thread Safety:** This class is **not thread-safe** and must be managed by the server's main thread. The execute method operates on live Entity-Component-System (ECS) data via the Store and Ref parameters, which are inherently unsafe for concurrent access. The pattern of queuing the final operation with BuilderToolsPlugin.addToQueue is a concurrency control mechanism, ensuring the actual world modification is processed in a controlled, sequential manner by the builder system.

## API Surface
The primary contract is the overridden execute method, invoked by the server's command handler.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ContractSelectionCommand() | constructor | O(1) | Registers the command name, description, permission level, and argument definitions with the superclass. |
| execute(context, store, ref, playerRef, world) | void | O(N) | Validates and processes the command. Complexity is O(N) where N is the number of axes provided, as it loops to create a task for each. |

## Integration Patterns

### Standard Usage
A developer does not typically interact with this class directly. The system is invoked via player input in the game client. The engine handles routing the command to this class.

A player triggers this command by typing it in chat:
```
# Contracts the selection by 10 blocks along the axis the player is facing
/contract 10

# Contracts the selection by 5 blocks along the world's X and Z axes
/contract 5 --axis X Z
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ContractSelectionCommand()` in game logic. The command is managed by the server's command registry. Creating a new instance will have no effect as it will not be registered to handle any input.
-   **Manual Invocation:** Do not call the `execute` method directly. This bypasses critical infrastructure, including permission checks, argument parsing, and context provision. The `CommandContext` and ECS parameters are complex objects that must be supplied by the engine's command handler.

## Data Pipeline
The flow of data for this command begins with player input and ends with a deferred task that modifies the world state.

> Flow:
> Player Chat Input -> Server Command Parser -> **ContractSelectionCommand.execute()** -> Argument & State Validation -> BuilderToolsPlugin.addToQueue(lambda) -> Builder System Worker -> World.contract() -> ECS State Change

