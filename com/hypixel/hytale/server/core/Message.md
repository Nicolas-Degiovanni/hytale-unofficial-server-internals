---
description: Architectural reference for Message
---

# Message

**Package:** com.hypixel.hytale.server.core
**Type:** Transient

## Definition
```java
// Signature
public class Message {
```

## Architecture & Concepts

The Message class is a high-level, fluent builder API for constructing complex, stylable, and translatable text components. It serves as the primary mechanism for creating any text intended for display to a user, whether in chat, UI elements, or game world entities.

Architecturally, this class acts as a facade over the raw data structure, FormattedMessage. While FormattedMessage is a plain data object suitable for serialization, Message provides the ergonomic, chainable interface for developers. Its core responsibility is to abstract the complexity of building a hierarchical message structure, managing internationalization (i18n) keys, and applying styling attributes.

The static Codec field is a critical component, integrating this class directly into the engine's serialization pipeline. This indicates that Message objects are designed to be transmittable over the network or persisted to disk, transforming the rich object structure into a compact BSON or JSON representation and back.

## Lifecycle & Ownership

-   **Creation:** Instances are created exclusively through static factory methods such as Message.raw, Message.translation, or Message.empty. Direct instantiation is disallowed. A new Message object is typically created for each distinct text component required by the application.

-   **Scope:** The lifecycle of a Message object is intentionally short. It is a transient object, designed to exist only for the duration of its construction and subsequent consumption by another system (e.g., a chat service, a UI renderer).

-   **Destruction:** The object is managed by the Java garbage collector. There are no manual cleanup or disposal methods. Once the Message has been passed to another system and all local references fall out of scope, it becomes eligible for garbage collection.

## Internal State & Concurrency

-   **State:** The Message class is highly **mutable**. Each call to a builder method like param, color, or insert modifies the internal state of the wrapped FormattedMessage object. The state is a complex object graph representing the message, its children, parameters, and styling.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. Its design as a fluent builder assumes single-threaded access for construction. Concurrent calls to its builder methods will result in race conditions and a corrupted internal state.

    **WARNING:** Confine Message object creation and modification to a single thread. Once built, it can be passed to other threads if treated as immutable, but further modification is unsafe.

## API Surface

The public API is designed as a fluent interface for building messages.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| translation(String messageId) | static Message | O(1) | Creates a new Message from an internationalization key. |
| raw(String message) | static Message | O(1) | Creates a new Message from a raw, untranslated string. |
| param(String key, T value) | Message | O(1) | Adds a substitution parameter for translation. Supports multiple primitive types. |
| color(String hex) | Message | O(1) | Sets the color of the message text. |
| bold(boolean bold) | Message | O(1) | Sets the bold styling property. |
| insert(Message child) | Message | O(N) | Appends a child Message, creating a composite message. Complexity is proportional to the current number of children due to array copying. |
| getAnsiMessage() | String | O(N) | Renders the message to a plain string, resolving translations and formatting parameters. This is a potentially expensive operation. |
| getFormattedMessage() | FormattedMessage | O(1) | Returns the underlying raw data object used for serialization. |

## Integration Patterns

### Standard Usage

The intended use is to chain factory and builder methods to construct a complete message, which is then passed to a consuming system, such as a player's chat sender.

```java
// How a developer should normally use this
Message welcomeMessage = Message.translation("server.welcome")
    .param("playerName", player.getName())
    .color("#FFFF00")
    .bold(true);

Message motd = Message.raw("Have a great day!");

player.sendMessage(Message.join(welcomeMessage, motd));
```

### Anti-Patterns (Do NOT do this)

-   **Instance Re-use:** Do not modify a Message object after it has been consumed by another system. Its mutable nature makes re-use error-prone. Always create a new instance for a new message.
-   **Concurrent Modification:** Never call builder methods on the same Message instance from multiple threads. This will lead to unpredictable behavior and data corruption.
-   **Direct Instantiation:** Do not use `new Message()`. The constructor is protected for a reason. Always use the static factory methods like `Message.raw()` or `Message.translation()`.

## Data Pipeline

The Message class is a key component in the server's text processing and communication pipeline.

> **Serialization Flow:**
> Developer Code -> **Message Builder** -> FormattedMessage DTO -> **Message.CODEC** -> BSON/JSON Payload -> Network Layer

> **Rendering Flow (Server-Side):**
> Developer Code -> **Message Builder** -> getAnsiMessage() -> I18nModule -> Formatted String -> Server Console / Log File

