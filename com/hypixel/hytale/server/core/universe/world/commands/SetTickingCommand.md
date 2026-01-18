---
description: Architectural reference for SetTickingCommand
---

# SetTickingCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands
**Type:** Transient

## Definition
```java
// Signature
public class SetTickingCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The SetTickingCommand is a concrete implementation of the Command Pattern, designed to operate within the server's command processing framework. Its sole responsibility is to expose a low-level administrative function—controlling a world's simulation tick—to an authorized invoker, such as a server administrator or an automated script.

This class acts as a safe, managed endpoint for modifying core world state. By extending AbstractWorldCommand, it integrates into a system that automatically provides it with the necessary execution context, including the target World instance and a communication channel back to the command sender. This design decouples the command's invoker from the World subsystem, ensuring that state changes are performed through a well-defined and validated API rather than direct, uncontrolled method calls.

The command is self-describing, registering its name ("setticking") and required arguments (a single boolean) with the command system upon instantiation. The framework then uses this metadata for parsing, validation, and help generation.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the server's central CommandRegistry during the bootstrap phase. The registry discovers all command implementations, likely through classpath scanning, and creates a single instance of each to be held for the server's lifetime.
- **Scope:** Singleton-like within the context of the CommandRegistry. A single instance persists for the entire server session.
- **Destruction:** The instance is dereferenced and eligible for garbage collection only when the server shuts down and the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** This class is stateless and effectively immutable after construction. Its only field, tickingArg, is a final reference to an argument definition object. The execute method operates exclusively on parameters supplied by the framework and does not modify any internal fields.
- **Thread Safety:** The class itself is inherently thread-safe due to its stateless design. However, the operations it performs on the World object are not guaranteed to be safe. The command framework must ensure that execute is invoked from a thread that has safe access to the World instance, typically the main server thread responsible for the game loop. Concurrent execution of this command on the same World instance would be a critical design flaw in the calling framework.

## API Surface
The public contract is fulfilled by overriding the execute method from its parent class. Direct invocation is an anti-pattern; the method is designed to be called exclusively by the command processing system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | Modifies the ticking state of the provided World. Sends a confirmation message via the CommandContext. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use by developers. Its integration is declarative: the class simply needs to exist within the project's classpath. The server's command system will automatically discover, instantiate, and register it.

The intended interaction is through a command-line interface, such as the server console or in-game chat:

```text
# Example user interaction
/setticking true
/setticking false
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new SetTickingCommand()`. Doing so creates an orphan object that is not registered with the command system and will never be executed.
- **Manual Execution:** Do not call the `execute` method directly. This bypasses critical framework functionality, including permission checks, argument parsing, and context acquisition. Bypassing the framework can lead to inconsistent state and NullPointerExceptions.

## Data Pipeline
The flow of data for this command begins with user input and terminates with a state change in the game world and a feedback message to the user.

> Flow:
> User Input (`/setticking true`) -> Server Network Listener -> Command Parser -> Command Registry (resolves "setticking" to SetTickingCommand) -> Argument Binder (parses "true" into a boolean) -> **SetTickingCommand.execute()** -> World.setTicking() -> CommandContext.sendMessage() -> Network Layer -> User Client

