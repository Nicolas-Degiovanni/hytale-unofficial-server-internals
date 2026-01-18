---
description: Architectural reference for WorldConfigSetSpawnDefaultCommand
---

# WorldConfigSetSpawnDefaultCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.worldconfig
**Type:** Transient

## Definition
```java
// Signature
public class WorldConfigSetSpawnDefaultCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The WorldConfigSetSpawnDefaultCommand is a concrete implementation of the Command Pattern, designed to encapsulate a single, atomic server operation. It exists within the server's broader Command System, which is responsible for parsing, routing, and executing commands from authorized sources like the server console or privileged players.

This command's sole responsibility is to mutate the configuration of a specific World instance. By invoking this command, an administrator resets the world's spawn mechanism to its default behavior. This is achieved by setting the world's spawn provider to null, signaling the game engine to fall back to its built-in spawn logic. It acts as a state-mutating endpoint within the command infrastructure, decoupled from the systems that invoke it.

### Lifecycle & Ownership
- **Creation:** A single instance of this class is instantiated by the server's Command System during the server bootstrap process. The system scans for command definitions and registers them for later execution.
- **Scope:** The registered command object persists for the entire server session, held within the Command System's registry. The execution itself is ephemeral; a new context is provided for each invocation.
- **Destruction:** The instance is eligible for garbage collection upon server shutdown when the Command System itself is dismantled.

## Internal State & Concurrency
- **State:** This class is fundamentally stateless. Its only member field is a static final constant used for localization. All necessary state, such as the target World and the execution CommandContext, is passed as arguments to the execute method. This design ensures that a single command instance can be reused for countless executions without risk of state corruption.
- **Thread Safety:** The class instance is inherently thread-safe due to its stateless nature. However, the operations performed within the execute method are **not** thread-safe. It directly mutates the state of a World object. The Command System is responsible for enforcing thread safety by dispatching world-specific commands onto a serial execution queue, typically the target world's main update thread.

## API Surface
The public contract is defined by its superclass, AbstractWorldCommand. The constructor is intended for framework use only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | Sets the spawn provider on the target WorldConfig to null. Logs the action and sends a confirmation message to the command issuer. |

## Integration Patterns

### Standard Usage
This command is not intended to be used directly in code. It is invoked by an end-user or an automated system through the server's command-line interface. The Command System handles parsing the input and dispatching it to this class.

```java
// This is a conceptual representation of how the command is invoked by the system.
// A server administrator would type a command like:
// /world config setspawn default

// The CommandSystem would then resolve and execute the command internally.
CommandSystem commandSystem = server.getCommandSystem();
CommandContext context = createFromConsoleOrPlayer(...);

// The system dispatches the command, which eventually calls the execute method.
commandSystem.dispatch(context, "world config setspawn default");
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WorldConfigSetSpawnDefaultCommand()`. The command is useless without being registered in the Command System, and creating instances provides no value.
- **Direct Invocation:** Never call the `execute` method directly. Bypassing the Command System circumvents critical infrastructure, including permission checks, argument parsing, and thread synchronization, which can lead to world state corruption or security vulnerabilities.

## Data Pipeline
As a command, this class represents a control flow rather than a data processing pipeline. It is an endpoint in a user-initiated workflow.

> Flow:
> User Input (Console/Chat) -> Command Parser -> **WorldConfigSetSpawnDefaultCommand** -> World.getWorldConfig() -> State Mutation (setSpawnProvider) -> Logger & CommandContext (Feedback)

