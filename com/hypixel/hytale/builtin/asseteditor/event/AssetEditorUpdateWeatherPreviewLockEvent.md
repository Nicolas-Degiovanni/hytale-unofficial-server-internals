---
description: Architectural reference for AssetEditorUpdateWeatherPreviewLockEvent
---

# AssetEditorUpdateWeatherPreviewLockEvent

**Package:** com.hypixel.hytale.builtin.asseteditor.event
**Type:** Transient Event Object

## Definition
```java
// Signature
public class AssetEditorUpdateWeatherPreviewLockEvent extends EditorClientEvent<Void> {
```

## Architecture & Concepts
The AssetEditorUpdateWeatherPreviewLockEvent is a simple, immutable message object used within the engine's event-driven architecture. It serves a highly specific purpose: to decouple the user interface from the underlying weather simulation system within the Asset Editor.

When a user interacts with a UI element to lock or unlock the weather preview (preventing it from changing automatically), that UI component does not directly call the weather simulation manager. Instead, it instantiates and dispatches this event onto an event bus. A dedicated listener, typically the weather preview controller, subscribes to this event type and reacts accordingly.

This pattern adheres to the principle of separation of concerns, ensuring that UI components are only responsible for broadcasting user intent, not for implementing the logic that fulfills that intent. The generic type parameter **Void** in its parent class, EditorClientEvent, signifies that this is a one-way notification; it does not carry a complex payload or expect a reply.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a UI event handler within the Asset Editor. This typically occurs in response to a direct user action, such as toggling a checkbox.
- **Scope:** Extremely short-lived and stateless. The object exists only for the brief moment it is being passed from the publisher to its subscribers on the event bus.
- **Destruction:** The event object is not managed or owned by any long-lived system. Once it has been processed by all subscribers, it has no remaining references and becomes eligible for immediate garbage collection.

## Internal State & Concurrency
- **State:** Immutable. The core state, the boolean **locked** flag, is declared as a final field and is set only once during construction. This is a critical design feature for event objects, as it guarantees that the message cannot be altered while in transit, preventing state corruption and race conditions between subscribers.
- **Thread Safety:** Inherently thread-safe due to its immutability. An instance can be safely read from any thread without requiring synchronization primitives. However, standard engine practice dictates that UI-related events like this are created and consumed on the main client thread.

## API Surface
The public contract is minimal, consisting of a single accessor to retrieve the event's payload.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isLocked() | boolean | O(1) | Returns the intended state of the weather preview lock. True signifies the preview should be locked; false signifies it should be unlocked. |

## Integration Patterns

### Standard Usage
This event should be created and immediately dispatched to the relevant event bus, typically one scoped to the active EditorClient instance.

```java
// Executed within a UI component's event listener
void onLockCheckboxToggled(boolean isChecked) {
    EditorClient editorContext = getActiveEditor(); // Obtain the current editor context

    // Instantiate and post the event to notify other systems
    var lockEvent = new AssetEditorUpdateWeatherPreviewLockEvent(editorContext, isChecked);
    editorContext.getEventBus().post(lockEvent);
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful References:** Do not store a reference to this event object within a listener or any other component. Its data should be consumed and acted upon immediately, then the object should be discarded.
- **Misinterpreting as a Command:** This event is a notification that a state change was requested. It is not a command that guarantees execution. Listeners are responsible for handling the logic.
- **Instantiation without Dispatch:** Creating an instance of AssetEditorUpdateWeatherPreviewLockEvent without posting it to an event bus is a no-op. The object will be created and then immediately garbage collected with no effect on the application state.

## Data Pipeline
The flow of data for this event is linear and unidirectional, originating from user input and terminating in a state change within a core editor system.

> Flow:
> User Input (UI Checkbox Toggle) -> UI Event Handler -> **AssetEditorUpdateWeatherPreviewLockEvent** (Instantiation) -> Editor-Scoped Event Bus -> WeatherPreviewController (Subscription) -> Weather Simulation State Update

