---
description: Architectural reference for EchoCommand
---

# EchoCommand

**Package:** com.hypixel.hytale.builtin.commandmacro
**Type:** Transient

## Definition
```java
// Signature
public class EchoCommand extends CommandBase {
```

## Architecture & Concepts
The EchoCommand is a concrete implementation of the **Command Pattern**, designed to be discovered and managed by the server's central command system. It encapsulates all logic required to define and execute a simple server-side command, `/echo`.

This class serves as a self-contained definition, specifying its name ("echo"), description, and required arguments. It inherits from CommandBase, which provides the necessary contract for the command system to execute it. Its primary role is to decouple the command invocation (a user typing in the chat) from the business logic (sending a message back to the user). It leverages the argument parsing framework via RequiredArg to declare its input requirements, allowing the framework to handle input validation and type conversion before execution begins.

### Lifecycle & Ownership
- **Creation:** Instantiated by the server's command registration service during the bootstrap phase. The system typically scans the classpath for all subclasses of CommandBase and creates a single instance of each to populate the command registry.
- **Scope:** The EchoCommand object is a stateless singleton for the command's definition. It persists for the entire server session. It does not hold data related to any specific execution; that state is passed in via the CommandContext.
- **Destruction:** The instance is de-referenced and eligible for garbage collection only when the server shuts down or the module containing it is unloaded.

## Internal State & Concurrency
- **State:** The EchoCommand instance is effectively **immutable** after its constructor completes. Its state, which includes the command name and the definition of its arguments (messageArg), does not change during its lifetime. All execution-specific data is managed externally within the CommandContext object.
- **Thread Safety:** This class is inherently **thread-safe**. Because it is stateless, the same instance can be safely used by multiple threads simultaneously. The command system guarantees that the `executeSync` method is invoked on a designated synchronous game thread, preventing race conditions related to game world modifications.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EchoCommand() | constructor | O(1) | Initializes the command's metadata, including its name and argument definitions. |
| executeSync(CommandContext) | void | O(N) | Executes the command's logic. Complexity is linear based on the length (N) of the input message due to the string replacement operation. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly. It is registered with the command system and triggered by user input. Programmatic invocation should be done through the command system's dispatch mechanism, not by direct method calls.

```java
// Example of the server's command system dispatching to this command
// A developer would typically not write this code.

CommandSystem commandSystem = server.getCommandSystem();
ServerPlayer sender = ...;

// This simulates a player typing "/echo Hello World"
commandSystem.dispatch(sender, "echo Hello World");
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new EchoCommand()` in game logic. The instance is managed by the command registry. Creating a new instance has no effect and will not be registered or executable.
- **Stateful Implementation:** Do not add mutable fields to this class to track execution state. For example, adding a `lastSender` field would create a severe race condition and is incorrect. All state must be retrieved from the `CommandContext` passed into `executeSync`.

## Data Pipeline
The flow of data for a typical `/echo` command execution is linear and synchronous.

> Flow:
> User Chat Input (`/echo "message"`) -> Network Packet -> Server Command Parser -> **EchoCommand** -> Message Object Creation -> Network Packet -> Client Chat Display

