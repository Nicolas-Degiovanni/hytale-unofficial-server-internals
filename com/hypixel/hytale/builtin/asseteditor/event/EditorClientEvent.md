---
description: Architectural reference for EditorClientEvent
---

# EditorClientEvent

**Package:** com.hypixel.hytale.builtin.asseteditor.event
**Type:** Transient (Abstract Base Class)

## Definition
```java
// Signature
public abstract class EditorClientEvent<KeyType> implements IEvent<KeyType> {
```

## Architecture & Concepts
The EditorClientEvent is an abstract base class that serves as the foundation for all events originating within the Asset Editor framework. It is not a concrete event itself, but rather a contract that ensures any event fired from an editor context carries a reference to its source, the EditorClient.

This class embodies the *Event-Carried State Transfer* pattern. Instead of requiring event listeners to query a global state manager to determine the context of an event, the context is embedded directly within the event payload. This design dramatically simplifies event handling logic, decouples listeners from the editor's internal state, and makes the system more robust, especially in scenarios where multiple editor instances might coexist.

By implementing the IEvent interface, all subclasses of EditorClientEvent are guaranteed to be compatible with the engine's core event bus for dispatch and subscription.

### Lifecycle & Ownership
-   **Creation:** Concrete subclasses are instantiated by an EditorClient instance or one of its sub-components in direct response to a user action or internal state change (e.g., saving a file, selecting a tool).
-   **Scope:** The object's lifetime is exceptionally short. An instance is created, dispatched onto the event bus, processed by all subscribers, and then immediately becomes eligible for garbage collection.
-   **Destruction:** There is no explicit destruction method. The Java Garbage Collector reclaims the memory once the event dispatch cycle is complete and no further references to the event object exist.

## Internal State & Concurrency
-   **State:** The state of an EditorClientEvent instance is **immutable**. The internal reference to the EditorClient is marked as final and is exclusively set during construction. The object acts as a read-only data carrier.
-   **Thread Safety:** The event object itself is inherently **thread-safe** due to its immutability and can be safely passed across thread boundaries.

    **WARNING:** While the event object is thread-safe, the EditorClient it references may not be. Consumers of this event must adhere to the threading guarantees of the EditorClient class itself. All interactions with the client obtained via getEditorClient must occur on the main thread unless explicitly documented otherwise.

## API Surface
The public contract is minimal, focusing solely on providing access to the event's source.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getEditorClient() | EditorClient | O(1) | Returns the non-null EditorClient instance that originated this event. |

## Integration Patterns

### Standard Usage
This class is not used directly. Developers must extend it to create a specific, concrete event type. Event listeners then subscribe to the concrete type and use the inherited getEditorClient method to retrieve the event's context.

```java
// In an event listener class, typically a system or UI controller.

@Subscribe
public void onSpecificAssetChange(ConcreteAssetChangeEvent event) {
    // Retrieve the source client directly from the event payload.
    EditorClient sourceClient = event.getEditorClient();

    // Use the client to update UI or perform other contextual actions.
    sourceClient.getUIManager().showNotification("Asset was changed!");
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** You cannot use `new EditorClientEvent()`. This is an abstract class and attempting to instantiate it will result in a compile-time error. Always create a concrete subclass.
-   **Long-Term Storage:** Do not cache or store event objects in collections or fields after they have been processed. They are transient data packets. Retaining them can cause memory leaks by preventing the EditorClient and its associated resources from being garbage collected.
-   **State Mutation:** Do not use reflection or other means to attempt to change the internal EditorClient reference after construction. The immutability of event objects is a core design principle.

## Data Pipeline
EditorClientEvent is a data payload that flows through the engine's event system. The typical flow for a concrete subclass is as follows:

> Flow:
> User Interaction (e.g., Click) -> EditorClient Logic -> **new ConcreteEditorEvent(this)** -> EventBus.post() -> Subscribers / Listeners -> UI or State Update

