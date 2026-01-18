---
description: Architectural reference for CommandException
---

# CommandException

**Package:** com.hypixel.hytale.server.core.command.system.exceptions
**Type:** Abstract Exception Base

## Definition
```java
// Signature
public abstract class CommandException extends RuntimeException {
```

## Architecture & Concepts
CommandException is the abstract base class for all user-facing exceptions within the server's command processing system. It establishes a critical contract for error handling that separates the failure condition from its presentation to the user.

Unlike a standard Java exception which is primarily for developers, a CommandException is designed to be caught and translated into a human-readable, localized message for the CommandSender (e.g., a player or a console). This architecture ensures a consistent and user-friendly error reporting mechanism across all server commands. Any logic that can fail during command parsing, validation, or execution *must* throw a derivative of this class to participate in this managed error-handling pipeline.

It fundamentally acts as a control-flow mechanism, allowing deep implementation details to signal a specific, reportable failure to the high-level command dispatcher.

### Lifecycle & Ownership
- **Creation:** Concrete subclasses are instantiated and thrown by the command parsing or execution logic when a recoverable, user-reportable error is detected. For example, a CommandSyntaxException is thrown if arguments are malformed.
- **Scope:** Highly transient. An instance of CommandException exists only for the duration of its propagation up the call stack until it is caught by the central command handler. Its lifespan is measured in microseconds.
- **Destruction:** The object is eligible for garbage collection immediately after its corresponding catch block completes execution. It holds no persistent state and is not referenced outside of the exception handling flow.

## Internal State & Concurrency
- **State:** As a RuntimeException, it inherently contains a stack trace and an optional message. Concrete implementations are expected to be immutable, capturing all necessary context for error reporting at the moment of their creation.
- **Thread Safety:** Exception objects are not designed for multi-threaded access. They are created, thrown, and caught within the confines of a single thread's execution. Sharing a CommandException instance across threads is a severe anti-pattern and will lead to unpredictable behavior. The class itself contains no synchronization primitives.

## API Surface
The public contract is defined by its single abstract method, which forces all subclasses to provide a mechanism for user notification.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| sendTranslatedMessage(CommandSender) | void | O(N) | Abstract method. Implementations must send a localized, user-friendly error message to the provided CommandSender. Complexity depends on the translation and network systems. |

## Integration Patterns

### Standard Usage
The intended pattern is to catch specific subclasses of CommandException within a global command execution wrapper. This allows for centralized, consistent error reporting to the user.

```java
// In a central command dispatcher
try {
    command.execute(sender, args);
} catch (CommandException ex) {
    // The exception itself knows how to report the error
    ex.sendTranslatedMessage(sender);
    // Optionally log the full stack trace for server admins
    log.warn("Command execution failed for {}: {}", sender.getName(), ex.getMessage());
}
```

### Anti-Patterns (Do NOT do this)
- **Catching Generic Exceptions:** Do not catch a generic RuntimeException or Exception when a CommandException is expected. Doing so bypasses the user-facing feedback system, leaving the user unaware of the failure.
- **Swallowing the Exception:** Catching a CommandException and failing to call sendTranslatedMessage defeats its entire purpose. The user who issued the command will receive no feedback.
- **Incorrect Instantiation:** This class is abstract and cannot be instantiated with *new CommandException()*. You must throw a concrete implementation like CommandSyntaxException or a custom-defined subclass.

## Data Pipeline
CommandException is part of a control-flow pipeline, not a data-processing pipeline. It represents a terminal state for a command's execution path.

> Flow:
> Command Execution Logic -> **throw new ConcreteCommandException()** -> Command Dispatcher Catch Block -> `exception.sendTranslatedMessage(sender)` -> Network Packet Sent to Client

