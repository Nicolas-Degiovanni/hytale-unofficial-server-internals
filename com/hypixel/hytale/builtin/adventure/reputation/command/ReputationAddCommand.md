---
description: Architectural reference for ReputationAddCommand
---

# ReputationAddCommand

**Package:** com.hypixel.hytale.builtin.adventure.reputation.command
**Type:** Transient Command Object

## Definition
```java
// Signature
public class ReputationAddCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The ReputationAddCommand class provides a server-side, executable command for administrators to modify a player's reputation score. It serves as a direct, imperative interface to the underlying ReputationPlugin, bridging the gap between user input (from the server console or an in-game entity) and the game's state management systems.

Architecturally, this class is a self-contained unit of work within the server's command processing framework. It extends AbstractTargetPlayerCommand, delegating the complex logic of parsing and resolving a target player to its parent class. This allows ReputationAddCommand to focus solely on its core responsibility: validating its specific arguments and applying the reputation change.

The command defines its required arguments internally: a ReputationGroup asset identifier and an integer value. This declarative approach allows the command system to automatically parse, validate, and provide typed arguments to the execution logic, significantly reducing boilerplate and error handling within the command itself.

## Lifecycle & Ownership
- **Creation:** A single instance of ReputationAddCommand is instantiated by the server's command registration system during the server bootstrap phase or when its parent plugin is loaded. It is not intended for manual instantiation by developers.
- **Scope:** The command object itself is a long-lived singleton that persists for the entire server session, acting as a stateless blueprint for execution. The CommandContext object passed to its execute method is ephemeral, existing only for the duration of a single command invocation.
- **Destruction:** The object is eligible for garbage collection when the server shuts down or the associated plugin is unloaded, at which point the command registry releases its reference.

## Internal State & Concurrency
- **State:** The ReputationAddCommand object is effectively immutable after construction. Its internal fields, which define the command's arguments, are final and are initialized once in the constructor. It holds no state related to any specific execution; all necessary data is provided via the CommandContext during the call to the execute method.
- **Thread Safety:** This class is thread-safe by immutability. However, the execute method performs mutations on the game world's state via the ReputationPlugin and the EntityStore. The server's command framework **must** ensure that the execute method is always invoked on the main game loop thread to prevent race conditions and data corruption. Direct invocation from asynchronous tasks is not supported and will lead to undefined behavior.

## API Surface
The public contract is defined by its constructor for framework instantiation and the overridden execute method which contains the core logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, ...) | protected void | Variable | Executes the reputation change. Complexity depends on the underlying EntityStore lookup. Throws exceptions if arguments are invalid or the target player cannot be resolved. |

## Integration Patterns

### Standard Usage
A developer does not call methods on this class directly. Instead, an instance is registered with the server's command system, typically during plugin initialization. The system then handles the entire lifecycle of parsing and execution.

```java
// Example of registering the command with a hypothetical registry
// This is typically done once when the server or plugin starts.

CommandRegistry registry = server.getCommandRegistry();
registry.register(new ReputationCommandGroup()
    .withSubCommand(new ReputationAddCommand())
    .withSubCommand(new ReputationRemoveCommand())
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Invocation:** Never create an instance of ReputationAddCommand and call the execute method manually. This bypasses critical framework services, including argument parsing, permission checks, and context setup, and will likely result in a NullPointerException or inconsistent game state.
- **Stateful Implementation:** Do not add mutable member variables to this class to track state between executions. Command objects must be stateless. All required state must be read from the CommandContext or the World.

## Data Pipeline
The flow of data for a typical invocation is managed entirely by the server command framework and the Hytale game engine.

> Flow:
> Raw Command String -> Command System Parser -> **ReputationAddCommand** (Argument Validation) -> `execute` method -> ReputationPlugin -> EntityStore (Player Component Mutation) -> CommandContext (Success Message) -> Command Source

---

