---
description: Architectural reference for GeneralCommandException
---

# GeneralCommandException

**Package:** com.hypixel.hytale.server.core.command.system.exceptions
**Type:** Transient / Data Transfer Object

## Definition
```java
// Signature
public class GeneralCommandException extends CommandException {
```

## Architecture & Concepts
The GeneralCommandException is a specialized, user-facing exception within the server's command processing framework. Its primary role is to terminate a command's execution flow and carry a structured, translatable error message back to the original command sender.

Unlike a standard Java exception which carries a simple string, this class encapsulates a rich **Message** object. This design decouples the command's business logic from the presentation of its error states. The command logic simply needs to identify an error condition, construct an appropriate **Message**, and throw this exception. A higher-level command dispatcher is then responsible for catching it and using the provided API to deliver a consistently formatted error response to the user.

The automatic coloring of the message to red within the constructor enforces a uniform look and feel for all general command errors presented to the client.

### Lifecycle & Ownership
- **Creation:** Instantiated and thrown on-demand by command handlers or argument parsers when a recoverable, user-attributable error occurs. For example, providing an invalid target or a malformed argument.
- **Scope:** Ephemeral. The object's lifetime is confined to the stack unwinding process during exception handling. It exists only from the point it is thrown to the point it is caught by the command execution engine.
- **Destruction:** The object becomes eligible for garbage collection immediately after the `catch` block that handles it completes. It holds no persistent state and is not referenced elsewhere.

## Internal State & Concurrency
- **State:** Effectively immutable. The internal **Message** field is assigned once during construction and is not modified thereafter. While the constructor invokes the **color** method on the provided **Message**, the reference itself is final.
- **Thread Safety:** This class is inherently thread-safe due to its immutable state. However, in practice, instances are created, thrown, and handled within the confines of a single thread responsible for processing a specific user's command. It is not designed to be shared across threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| GeneralCommandException(Message) | constructor | O(1) | Creates an exception, wrapping the provided message and coloring it red. |
| sendTranslatedMessage(CommandSender) | void | O(1) | Fulfills the parent contract by sending the encapsulated message to the sender. |
| getMessageText() | String | O(1) | Returns a string representation of the error, primarily for logging. It attempts to get a formatted ANSI message before falling back to raw text or the message ID. |

## Integration Patterns

### Standard Usage
This exception is intended to be caught by a centralized command dispatcher, which then forwards the contained message to the command's origin.

```java
// Inside a central command execution loop
try {
    command.execute(sender, args);
} catch (GeneralCommandException e) {
    // The exception itself knows how to deliver the error message
    e.sendTranslatedMessage(sender);
    // Log the failure for server diagnostics
    log.warn("Command failed with user error: " + e.getMessageText());
}
```

### Anti-Patterns (Do NOT do this)
- **Swallowing the Exception:** Catching this exception without calling **sendTranslatedMessage** is an anti-pattern. This will result in the command failing silently, providing no feedback to the user.
- **Using for Internal Errors:** This exception is for user-facing errors. For critical, non-recoverable server bugs or system failures, a standard **RuntimeException** should be thrown to ensure it is handled by a higher-level error boundary and logged appropriately.
- **Empty Messages:** Constructing a **GeneralCommandException** with a null or empty **Message** object will lead to unpredictable behavior or a **NullPointerException**.

## Data Pipeline
The primary function of this class is to act as a carrier in an error-handling data flow. It transforms a logical error into a user-visible message.

> Flow:
> Command Logic Failure -> **new GeneralCommandException(message)** -> Thrown to Stack -> Command Dispatcher `catch` block -> **exception.sendTranslatedMessage(sender)** -> Network Layer -> Client UI
---

