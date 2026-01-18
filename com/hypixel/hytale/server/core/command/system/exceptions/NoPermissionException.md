---
description: Architectural reference for NoPermissionException
---

# NoPermissionException

**Package:** com.hypixel.hytale.server.core.command.system.exceptions
**Type:** Transient

## Definition
```java
// Signature
public class NoPermissionException extends CommandException {
```

## Architecture & Concepts
The NoPermissionException is a specialized, checked exception within the server's command processing framework. It represents a specific, recoverable failure state: a CommandSender attempting to execute a command without possessing the required permission node.

Architecturally, this class serves as a critical control flow mechanism. It decouples the *detection* of a permission failure from the *reporting* of that failure. Low-level permission checking logic can simply throw this exception, confident that a higher-level handler in the command execution pipeline will catch it and present a standardized, user-friendly message.

This design adheres to the "Tell, Don't Ask" principle. The exception object itself encapsulates the logic for generating its own translated error message via the sendTranslatedMessage method. The catching code does not need to inspect the exception's internal state; it simply instructs the exception to report itself to the originating user. This centralizes error message formatting and ensures a consistent user experience for all permission-related errors.

### Lifecycle & Ownership
- **Creation:** Instantiated and thrown by the core command system's permission validation layer when a check against a CommandSender's permissions fails. It is not intended for manual instantiation in typical command logic.
- **Scope:** Extremely short-lived. Its lifecycle is confined to the call stack of a single command execution, from the point it is thrown to the point it is caught by the global command exception handler.
- **Destruction:** The object is eligible for garbage collection immediately after the `catch` block completes its execution. It holds no persistent state or system resources.

## Internal State & Concurrency
- **State:** Immutable. The required permission string is stored in a final field, set once during construction. The object's state cannot be modified after creation, making its behavior entirely predictable.
- **Thread Safety:** Inherently thread-safe due to its immutability. While it can be safely passed between threads, its use case is almost exclusively confined to the single server thread responsible for processing a given command.

## API Surface
The primary public contract is the method used by the framework to report the error to the user.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| sendTranslatedMessage(sender) | void | O(1) | Constructs and sends a localized error message to the specified CommandSender. The message is colored red and includes the specific permission node that was denied. |

## Integration Patterns

### Standard Usage
This exception is not typically used directly. Instead, developers interact with the system that throws it. The standard pattern is to rely on a higher-level command handler to catch this specific exception type and correctly report the failure to the user.

```java
// A simplified view of the framework's central command handler
try {
    command.execute(sender, args);
} catch (NoPermissionException e) {
    // The exception itself knows how to report the error
    e.sendTranslatedMessage(sender);
} catch (CommandException other) {
    // Handle other command-related failures
    other.sendTranslatedMessage(sender);
}
```

### Anti-Patterns (Do NOT do this)
- **Swallowing The Exception:** Catching a NoPermissionException and failing to call sendTranslatedMessage is a critical error. This silences a user-facing failure, leaving the user confused as to why a command did not work.
- **Manual Message Formatting:** Do not catch this exception only to extract the permission string and build a custom error message. This bypasses the server's centralized translation and message formatting system, leading to inconsistent UI and breaking localization.

## Data Pipeline
NoPermissionException acts as a signal in a control flow pipeline rather than a data transformation pipeline. It aborts the normal execution flow and redirects it to an error handling path.

> Flow:
> Command Input -> Command Dispatcher -> Permission Check -> **NoPermissionException Thrown** -> Framework Catch Block -> `sendTranslatedMessage` -> Formatted Network Packet to Client

