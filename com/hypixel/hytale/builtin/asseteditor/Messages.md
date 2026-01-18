---
description: Architectural reference for Messages
---

# Messages

**Package:** com.hypixel.hytale.builtin.asseteditor
**Type:** Utility / Constants

## Definition
```java
// Signature
public class Messages {
```

## Architecture & Concepts
The Messages class is a centralized, compile-time constant provider for the server-side Asset Editor subsystem. It serves as a manifest for translatable string keys, decoupling the core logic of the asset editor from the specific user-facing text. This pattern is critical for internationalization (i18n) and maintainability, ensuring that all user-facing error and status messages are sourced from a single, verifiable location.

This class does not contain any logic. Its sole purpose is to hold static final references to Message objects. These objects encapsulate the translation keys, which are later resolved by the server's localization engine into the appropriate language for a given player's client. It acts as a type-safe enumeration of possible feedback messages that the asset editor can produce.

## Lifecycle & Ownership
- **Creation:** The Messages class is loaded by the Java Virtual Machine's ClassLoader when it is first referenced by another component, typically an Asset Editor command handler or validation service. The static Message fields are instantiated and initialized once during this class-loading phase.
- **Scope:** Application-level. Once loaded, the class and its static members persist for the entire lifetime of the server process.
- **Destruction:** The class is unloaded when the application's ClassLoader is garbage collected, which generally only occurs during a full server shutdown.

## Internal State & Concurrency
- **State:** This class is stateless. All fields are declared as **static final**, making their references immutable after class initialization. The underlying Message objects are themselves immutable value objects.
- **Thread Safety:** This class is inherently thread-safe. As a container for compile-time constants initialized during class loading, there are no mutable states and therefore no opportunities for race conditions. It can be safely accessed from any thread without synchronization.

## API Surface
The public contract consists exclusively of static fields. Access is a direct field read operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| USAGE_DENIED_MESSAGE | Message | O(1) | Message key for when a player lacks permission to use the asset editor. |
| INVALID_FILENAME_MESSAGE | Message | O(1) | Message key for when a provided filename contains invalid characters or format. |
| OUTSIDE_ASSET_ROOT_MESSAGE | Message | O(1) | Message key for attempts to access a directory outside the allowed asset pack root. |
| UNKNOWN_ASSETPACK_MESSAGE | Message | O(1) | Message key for when a specified asset pack cannot be found. |

## Integration Patterns

### Standard Usage
This class is intended to be used for retrieving pre-defined Message objects to be sent to players. The server's messaging or command framework will then use this object to perform the necessary translation lookups.

```java
// Example from a hypothetical command handler
import com.hypixel.hytale.server.core.Player;
import com.hypixel.hytale.builtin.asseteditor.Messages;

public void handleAssetCommand(Player player, String[] args) {
    if (!player.hasPermission("hytale.asseteditor.use")) {
        player.sendMessage(Messages.USAGE_DENIED_MESSAGE);
        return;
    }
    // ... further command logic
}
```

### Anti-Patterns (Do NOT do this)
- **Hard-Coding Keys:** Do not use the raw translation keys like "server.assetEditor.messages.usageDenied" directly in the code. Always reference the constants provided by this class to prevent errors from key changes and to maintain a centralized reference.
- **Attempting Modification:** The fields are final. Any attempt to modify them using reflection will break the immutability contract of the server's messaging system and result in undefined behavior.

## Data Pipeline
The Messages class does not process data; it is a source of static data. The constants it provides are injected into a wider data flow for player communication.

> Flow:
> Asset Editor Service (Validation Logic) -> **Messages.INVALID_FILENAME_MESSAGE** -> Server Message Emitter -> Network Layer -> Client Localization Engine -> Rendered UI Text

