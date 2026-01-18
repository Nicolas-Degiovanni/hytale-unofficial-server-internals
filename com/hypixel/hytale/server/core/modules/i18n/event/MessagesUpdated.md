---
description: Architectural reference for MessagesUpdated
---

# MessagesUpdated

**Package:** com.hypixel.hytale.server.core.modules.i18n.event
**Type:** Transient

## Definition
```java
// Signature
public class MessagesUpdated implements IEvent<Void> {
```

## Architecture & Concepts
The MessagesUpdated class is an immutable event object that functions as a data transfer object (DTO) within the server's event bus system. It is a fundamental component of the dynamic internationalization (i18n) module, designed to signal that localization strings have been updated, added, or removed at runtime.

Its primary architectural purpose is to decouple the source of a localization change (e.g., a file system watcher, a live-ops command) from the various consumers that need to react to it (e.g., player session handlers, UI systems). By broadcasting this event, the i18n module can notify the entire server of a change without maintaining direct references to listeners, promoting a clean, event-driven architecture.

The data is structured as a nested map: `Map<Locale, Map<MessageKey, MessageValue>>`. This allows a single event to efficiently carry updates for multiple languages simultaneously.

## Lifecycle & Ownership
- **Creation:** An instance of MessagesUpdated is created exclusively by the central i18n service when it detects a change in the underlying message sources. This is an internal system-level operation.
- **Scope:** This object is extremely short-lived. Its scope is confined to the duration of its dispatch through the event bus. It is created, broadcast to all subscribers, and then immediately becomes eligible for garbage collection.
- **Destruction:** The Java Garbage Collector reclaims the object's memory once the event dispatch is complete and no strong references remain. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** Immutable. The `changedMessages` and `removedMessages` fields are final and are populated only at construction. This design guarantees that the event's payload cannot be altered after it has been fired, ensuring all listeners receive the exact same data.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. It can be safely passed across thread boundaries, for example, from a background file-monitoring thread to the main server thread via the event bus, without any need for locks or other synchronization primitives.

## API Surface
The public API is minimal, providing read-only access to the event payload.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getChangedMessages() | Map | O(1) | Returns the complete map of new or modified messages, keyed by locale. |
| getRemovedMessages() | Map | O(1) | Returns the complete map of messages that were removed, keyed by locale. |

## Integration Patterns

### Standard Usage
The intended pattern for interacting with this class is to consume it within a service that subscribes to the server's event bus. Developers should never instantiate this class directly.

```java
// Example of a service listening for localization updates
public class PlayerNotificationService {

    @Subscribe
    public void onMessagesUpdated(MessagesUpdated event) {
        // Check for changes in the primary server language
        Map<String, String> englishUpdates = event.getChangedMessages().get("en_us");

        if (englishUpdates != null) {
            // Refresh cached messages or notify relevant systems
            System.out.println("Received " + englishUpdates.size() + " message updates.");
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new MessagesUpdated()`. This event is owned and generated exclusively by the core i18n module. Manually firing a synthetic event will lead to state desynchronization across the server.
- **Payload Modification:** Do not attempt to modify the maps returned by `getChangedMessages` or `getRemovedMessages`. While the collections are not wrapped in an unmodifiable view for performance reasons, altering their state is a severe violation of the immutable contract. Doing so will cause unpredictable and difficult-to-debug side effects in other event listeners.

## Data Pipeline
The MessagesUpdated event is a critical link in the server's live localization update pipeline. It translates a low-level resource change into a high-level, system-wide notification.

> Flow:
> External Source Change (e.g., file save, database update) -> I18n Resource Monitor -> **MessagesUpdated** (Instantiation) -> Server Event Bus -> Subscribed Listeners (e.g., Session Handlers, Command Parsers)

