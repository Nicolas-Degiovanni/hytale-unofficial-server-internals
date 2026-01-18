---
description: Architectural reference for AssetEditorActivateButtonEvent
---

# AssetEditorActivateButtonEvent

**Package:** com.hypixel.hytale.builtin.asseteditor.event
**Type:** Transient

## Definition
```java
// Signature
public class AssetEditorActivateButtonEvent extends EditorClientEvent<String> {
```

## Architecture & Concepts
The AssetEditorActivateButtonEvent is a simple, immutable message object used within the engine's event-driven architecture. Its sole purpose is to signal that a user has activated a specific UI button within the Asset Editor.

This class is a concrete implementation of the Command pattern. It encapsulates a user's request—activating a button—as an object. This decouples the event producer (the UI input system) from the event consumers (systems that need to react to the button press). By extending EditorClientEvent, it standardizes its structure, ensuring that any listener can identify the source editor instance that generated the event.

The event carries a single piece of data: the buttonId. This string identifier is the contract between the UI layout and the backend systems, allowing for flexible and data-driven UI interactions without hard-coding direct method calls.

## Lifecycle & Ownership
- **Creation:** Instantiated by the Asset Editor's internal UI framework in direct response to a user input event, such as a mouse click on a button element.
- **Scope:** Extremely short-lived. An instance of this event exists only for the duration of its dispatch through the event bus. It is a fire-and-forget message.
- **Destruction:** The object becomes eligible for garbage collection as soon as the event bus has finished notifying all subscribed listeners. No system should retain a long-term reference to an event object.

## Internal State & Concurrency
- **State:** Immutable. The internal state, specifically the buttonId, is declared as final and is set only once during construction. This guarantees that the event's data cannot be mutated during its transit through the system.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. It can be safely passed across thread boundaries without requiring locks or other synchronization primitives.

**WARNING:** While the event object itself is thread-safe, the listeners that consume it are responsible for their own thread safety. Handlers must be designed to execute safely if the event bus dispatches on multiple threads.

## API Surface
The public contract is minimal, focusing on data retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getButtonId() | String | O(1) | Retrieves the unique identifier of the button that was activated. |
| getEditorClient() | EditorClient | O(1) | *Inherited.* Returns the editor instance that originated this event. |

## Integration Patterns

### Standard Usage
This class is not meant to be manually instantiated or managed by most game code. Instead, systems subscribe to it via the engine's event bus. The standard pattern is to create a listener method that accepts this event as its parameter.

```java
// Example of a system listening for a specific button press
public class AssetEditorToolbarSystem {

    @Subscribe
    public void onButtonActivation(AssetEditorActivateButtonEvent event) {
        if ("save_asset_button".equals(event.getButtonId())) {
            // Logic to save the current asset
            performSaveOperation(event.getEditorClient());
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Reusing Instances:** Do not cache and reuse an event object. A new AssetEditorActivateButtonEvent must be instantiated for every single button activation to ensure data integrity.
- **Manual Dispatch:** Avoid passing event instances directly to other objects. Always publish events to the central event bus to respect the decoupled architecture. Bypassing the bus can lead to missed notifications and unpredictable system behavior.
- **Overloading with Logic:** An event should be a plain data carrier. Do not add business logic, validation, or processing methods to this class.

## Data Pipeline
The flow of this event through the engine is linear and unidirectional, originating from user input and terminating in system action.

> Flow:
> User Input (Mouse Click) -> UI Input Handler -> **AssetEditorActivateButtonEvent Instantiation** -> Engine Event Bus -> Subscribed Listeners (e.g., ToolbarSystem, AssetIOManager) -> Business Logic Execution

