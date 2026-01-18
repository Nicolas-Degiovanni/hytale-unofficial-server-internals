---
description: Architectural reference for ToolArgException
---

# ToolArgException

**Package:** com.hypixel.hytale.server.core.asset.type.buildertool.config.args
**Type:** Transient

## Definition
```java
// Signature
public class ToolArgException extends Exception {
```

## Architecture & Concepts
ToolArgException is a specialized, checked exception used to signal failures during the parsing and validation of arguments supplied to server-side builder tools. Its primary architectural role is to encapsulate a user-facing, localizable error message within an exception, separating error signaling from error presentation.

Unlike a standard Exception that carries a simple string, this class holds a structured **Message** object. This design is critical for the server's command framework. It allows low-level argument parsers to fail with a precise, translatable error key, which can then be caught by a higher-level command handler. The handler can then use the server's localization system to render the appropriate error message to the user in their configured language.

This class acts as a formal contract for error handling within the builder tool configuration system, ensuring that all argument-related failures provide rich, actionable feedback to the end-user.

### Lifecycle & Ownership
- **Creation:** Instantiated and thrown exclusively by argument parsing or validation logic when it encounters malformed, invalid, or missing input. The creator is responsible for constructing a valid, non-null Message object that accurately describes the failure.
- **Scope:** The lifecycle of a ToolArgException instance is extremely brief. It exists only for the duration of stack unwinding, from the point it is thrown until it is handled by a corresponding `catch` block.
- **Destruction:** The object becomes eligible for garbage collection immediately after the `catch` block that handles it completes. No manual resource management is required.

## Internal State & Concurrency
- **State:** The internal state of this class is **Immutable**. The core `translationMessage` field is declared `final` and is initialized only once within the constructor. The state inherited from the parent Exception class, such as the stack trace, is also populated at creation time and is not subsequently modified.
- **Thread Safety:** ToolArgException is inherently **thread-safe** due to its immutability. An instance can be safely passed across threads, although its typical use case is confined to the call stack of a single thread. No synchronization mechanisms are necessary.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ToolArgException(Message) | Constructor | O(1) | Constructs a new exception with a specified translation message. |
| ToolArgException(Message, Throwable) | Constructor | O(1) | Constructs a new exception with a message and a root cause. |
| getTranslationMessage() | Message | O(1) | Retrieves the structured, localizable message describing the error. |

## Integration Patterns

### Standard Usage
The intended pattern is to throw this exception from deep within a parsing system and catch it at the command execution boundary. The `catch` block is then responsible for translating the encapsulated Message and relaying it to the command's source (e.g., a player).

```java
// In a command handler
try {
    ToolArguments args = argumentParser.parse(rawInput);
    // ... execute tool logic with args
} catch (ToolArgException e) {
    // Retrieve the structured message
    Message errorMessage = e.getTranslationMessage();
    
    // Send the localized error back to the player
    player.sendMessage(localizationService.translate(errorMessage));
}
```

### Anti-Patterns (Do NOT do this)
- **Swallowing the Exception:** Catching this exception and failing to notify the user is a critical error. The user will receive no feedback for their invalid command input.
- **Ignoring the Message Object:** Do not rely on `e.getMessage()` for user-facing output. This returns a raw string and bypasses the entire localization system. Always use `e.getTranslationMessage()` to retrieve the object intended for localization.
- **Generic Catch Blocks:** Avoid catching the generic `Exception` when a ToolArgException is expected. This can mask other unexpected runtime errors and prevents specific handling of the user-facing error message.

## Data Pipeline
The primary function of this class is to carry error data from a point of failure to a point of user notification.

> Flow:
> Player Command Input -> Server Command Parser -> **ToolArgException (thrown)** -> Command Executor (caught) -> Localization Service -> Network Packet -> Client UI Error Message

---

