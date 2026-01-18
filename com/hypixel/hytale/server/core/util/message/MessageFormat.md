---
description: Architectural reference for MessageFormat
---

# MessageFormat

**Package:** com.hypixel.hytale.server.core.util.message
**Type:** Utility

## Definition
```java
// Signature
public final class MessageFormat {
```

## Architecture & Concepts
MessageFormat is a stateless, static utility class designed to enforce consistent and localized text formatting for common server messages. It acts as a centralized factory for creating complex, composite Message objects from primitive data types or collections.

The primary architectural role of this class is to decouple application logic from presentation formatting. Instead of business logic components manually constructing user-facing strings with hardcoded separators or layout rules, they delegate this responsibility to MessageFormat. This ensures that all lists, status indicators, and other recurring message patterns have a uniform appearance and behavior across the entire server application.

The class contains pre-computed, static instances for high-frequency messages like *enabled* and *disabled* states, avoiding repeated object allocation and translation lookups for these common cases.

**WARNING:** This class is a foundational component for server-to-client communication. Modifications to its formatting logic will have a widespread impact on how information is displayed to players.

## Lifecycle & Ownership
- **Creation:** As a final class with only static members and an implicit private constructor, MessageFormat is never instantiated. The Java ClassLoader loads it into memory upon first reference. Its static fields, ENABLED and DISABLED, are initialized once during this class-loading phase.
- **Scope:** Application-level. The class and its static methods persist for the entire lifetime of the server JVM process.
- **Destruction:** The class is unloaded by the JVM when the server application terminates. No manual resource management is required.

## Internal State & Concurrency
- **State:** MessageFormat is entirely stateless. Its methods are pure functions whose output depends solely on their input arguments. The static fields are immutable constants initialized at class-load time.
- **Thread Safety:** This class is inherently thread-safe. Its methods do not access or modify any shared mutable state. It can be safely invoked from any number of concurrent threads without synchronization or risk of data corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| enabled(boolean b) | Message | O(1) | Returns a pre-cached, colored Message for an enabled or disabled state. |
| list(Message header, Collection<Message> values) | Message | O(N) | Constructs a composite Message from a collection. The formatting intelligently adapts, using an inline style for small lists (<=4 items) and a multi-line format for larger ones. |

## Integration Patterns

### Standard Usage
This utility should be used whenever a command or system needs to present a list of items or a simple boolean state to a user. It abstracts away the specific translation keys and formatting logic.

```java
// How a developer should normally use this
Collection<Message> playerNames = getOnlinePlayers().stream()
    .map(player -> Message.raw(player.getName()))
    .collect(Collectors.toList());

Message header = Message.translation("server.commands.list.playerHeader");
Message formattedList = MessageFormat.list(header, playerNames);

// The formattedList is now ready to be sent to a client.
```

### Anti-Patterns (Do NOT do this)
- **Manual Formatting:** Do not re-implement list formatting logic within other services. This leads to inconsistent UI and localization difficulties. Always defer to MessageFormat to ensure a uniform user experience.
- **Attempting Instantiation:** The class cannot be instantiated with `new MessageFormat()`. All access must be through its static methods, for example, `MessageFormat.list(...)`.

## Data Pipeline
MessageFormat functions as a transformation step in the data-to-presentation pipeline. It takes structured server-side data and converts it into a structured, presentable Message object, which is then typically serialized and sent to the client for rendering.

> Flow:
> Raw Data (e.g., `boolean`, `Collection<String>`) -> Business Logic -> **MessageFormat** -> Composite `Message` Object -> Network Subsystem -> Client UI Renderer

