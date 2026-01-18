---
description: Architectural reference for SpawnSetDefaultCommand
---

# SpawnSetDefaultCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.teleport
**Type:** Singleton

## Definition
```java
// Signature
public class SpawnSetDefaultCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The SpawnSetDefaultCommand is a server-side command processor that implements a specific administrative action: resetting a world's spawn point to its default behavior. As a subclass of AbstractWorldCommand, it integrates directly into the server's command system, which is responsible for parsing and dispatching commands entered by players or through the server console.

Architecturally, this class embodies the Command Pattern. It encapsulates a single, atomic operation—modifying the WorldConfig—into a standalone object. The command system maintains a registry of these command objects, and upon matching a user's input, it invokes the command's execution logic.

This command is a "leaf" in a hierarchical command structure, identified by the name "default". It is designed to be a sub-command of a broader "spawn set" command, providing a specific mutation for world state. Its sole responsibility is to interact with the WorldConfig object to nullify the custom spawn provider, thereby reverting to the engine's default player spawning logic.

## Lifecycle & Ownership
- **Creation:** A single instance of SpawnSetDefaultCommand is created by the Command System during server initialization or plugin loading. The system discovers and registers all command implementations to build its command tree.
- **Scope:** The instance is a singleton managed by the Command System's registry. It persists for the entire server session.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection only when the server shuts down or the parent plugin is unloaded, at which point the Command System clears its registry.

## Internal State & Concurrency
- **State:** This class is **stateless**. Its only instance fields are static final constants, which are immutable. All operations are performed on external state objects, primarily the World and WorldConfig, which are passed as arguments to the execute method.
- **Thread Safety:** The class instance is inherently thread-safe due to its stateless design. However, the execution context is critical. The `execute` method is designed to be called by the server's main command processing loop, which must guarantee that all modifications to a given World object occur on that world's primary thread. Direct, multi-threaded invocation of the `execute` method would lead to severe race conditions within the WorldConfig.

## API Surface
The public contract is defined by its parent, AbstractWorldCommand.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | Resets the spawn provider in the given world's configuration to null. This is the primary entry point, invoked by the command system. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is exclusively managed and executed by the server's command framework. A user with sufficient permissions triggers its execution by typing the corresponding command in-game or in the console.

*Example User Interaction:*
```
/spawn set default
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new SpawnSetDefaultCommand()`. The command will not be registered with the server and will have no effect.
- **Manual Execution:** Calling the `execute` method directly from other parts of the codebase is a severe anti-pattern. This bypasses the entire command system's infrastructure, including critical permission checks, argument validation, and thread safety guarantees provided by the command dispatcher.

## Data Pipeline
This component acts as an endpoint in a control flow rather than a stage in a data processing pipeline. The flow is initiated by user input and results in a state change and a feedback message.

> Flow:
> User Command Input -> Server Network Layer -> Command Parser -> Command Dispatcher -> **SpawnSetDefaultCommand.execute()** -> WorldConfig State Mutation -> Confirmation Message -> User Client

