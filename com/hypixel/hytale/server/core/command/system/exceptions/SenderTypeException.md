---
description: Architectural reference for SenderTypeException
---

# SenderTypeException

**Package:** com.hypixel.hytale.server.core.command.system.exceptions
**Type:** Transient

## Definition
```java
// Signature
public class SenderTypeException extends CommandException {
```

## Architecture & Concepts
The SenderTypeException is a specialized, semantic exception within the server's command processing framework. Its primary role is to enforce type constraints on the entity executing a command. For example, if a command is designed to be executed only by a Player, this exception is thrown when the server console or another non-player entity attempts to invoke it.

This class is a key component of the framework's user-facing error reporting system. By extending CommandException, it integrates into a centralized error handling mechanism within the main command dispatcher. When caught, the dispatcher uses the polymorphic `sendTranslatedMessage` method to provide a clear, localized error message to the original sender, abstracting the details of message formatting away from individual command implementations.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand within a command handler's logic when a runtime check of the CommandSender's type fails. It is created specifically to be thrown.
- **Scope:** Ephemeral. The object's lifecycle is confined to the stack during the unwinding process of a single failed command execution. It exists only from the point it is thrown to the point it is caught and handled by the command dispatcher.
- **Destruction:** The object becomes eligible for garbage collection immediately after the dispatcher's `catch` block completes execution. There are no persistent references to it.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state, which consists of the expected sender type, is captured once at construction via a final field. The object cannot be modified after creation.
- **Thread Safety:** This class is inherently **Thread-Safe** due to its immutability. While command execution is typically handled on a single thread, this design ensures that the exception object could be safely passed across threads without risk of data corruption, though such a use case is not standard.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SenderTypeException(Class<?>) | constructor | O(1) | Constructs a new exception, capturing the required sender type that was violated. |
| sendTranslatedMessage(CommandSender) | void | O(1) | Sends a pre-defined, localized error message to the sender, informing them of the type mismatch. |

## Integration Patterns

### Standard Usage
The canonical use of this exception is to guard command logic that depends on a specific sender type. The command handler performs a check and throws the exception, delegating the responsibility of user notification to the framework.

```java
// Inside a command handler's execution logic
public void execute(CommandSender sender, String[] args) throws CommandException {
    // This command can only be run by a Player entity.
    if (!(sender instanceof Player)) {
        throw new SenderTypeException(Player.class);
    }

    Player player = (Player) sender;
    // ... continue with Player-specific logic
}
```

### Anti-Patterns (Do NOT do this)
- **Catching and Swallowing:** Never `try-catch` this exception within the command handler itself. Doing so subverts the centralized error handling mechanism and will prevent the user from receiving feedback on why their command failed.
- **Manual Message Sending:** Do not instantiate this class simply to call its `sendTranslatedMessage` method. The object is designed to be *thrown*. The framework is responsible for catching it and invoking the message sending logic.

## Data Pipeline
The SenderTypeException acts as a control flow signal that carries error context. It interrupts the normal execution path and redirects control to a handler that translates its state into a user-facing message.

> Flow:
> Command Input -> Command Dispatcher -> Command Handler Logic -> Runtime Type Check Fails -> **throw SenderTypeException** -> Command Dispatcher `catch` block -> `sendTranslatedMessage` called on exception object -> Formatted Message -> Network Layer -> Client receives error message

