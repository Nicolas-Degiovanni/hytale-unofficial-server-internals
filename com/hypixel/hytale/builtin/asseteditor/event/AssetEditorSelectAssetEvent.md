---
description: Architectural reference for AssetEditorSelectAssetEvent
---

# AssetEditorSelectAssetEvent

**Package:** com.hypixel.hytale.builtin.asseteditor.event
**Type:** Transient / Event

## Definition
```java
// Signature
public class AssetEditorSelectAssetEvent extends EditorClientEvent<Void> {
```

## Architecture & Concepts
The AssetEditorSelectAssetEvent is a message-based data structure, not a service. It functions as an immutable Data Transfer Object (DTO) within the Asset Editor's event-driven architecture. Its primary role is to signal a state change in the user interface, specifically when the user selects a different asset for editing.

This class is fundamental to decoupling the components of the Asset Editor. The UI element that triggers the selection (e.g., an asset tree view) does not need to know which other components (e.g., a 3D preview pane, a property inspector) need to react. By dispatching this event onto an event bus, the source of the action is fully decoupled from its consumers.

The event's design, which includes both the *newly selected* and *previously selected* asset paths, is deliberate. It provides subscribers with all necessary context to manage their state transitions cleanly. For example, a previewer can use the previous asset information to unload resources before loading the new ones, preventing memory leaks and invalid state. The generic type `Void` in its parent class `EditorClientEvent<Void>` signifies that this is a one-way notification; it does not expect a response from its listeners.

### Lifecycle & Ownership
-   **Creation:** Instantiated by a UI listener within the Asset Editor's view layer when a user interaction results in a new asset selection. The creating listener is responsible for providing the context of both the new and previous selections.
-   **Scope:** Extremely short-lived. An instance of this event exists only for the duration of its dispatch and processing by all subscribers on the event bus.
-   **Destruction:** The object becomes eligible for garbage collection immediately after the event bus has finished notifying all listeners. No system should retain a long-term reference to an event object.

## Internal State & Concurrency
-   **State:** **Immutable**. All member fields are declared `final` and are initialized exclusively through the constructor. This is a critical design feature for event objects, as it guarantees that the event's data cannot be mutated as it passes through various subscribers, some of which may operate on different threads.
-   **Thread Safety:** **Inherently thread-safe**. Due to its immutability, a single instance can be safely read by multiple threads without any need for external synchronization or locks. The responsibility for thread safety lies with the subscribers that consume the event, not the event object itself.

## API Surface
The public contract is minimal, consisting only of accessors for its immutable state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetType() | String | O(1) | Retrieves the type identifier for the newly selected asset. |
| getAssetFilePath() | AssetPath | O(1) | Retrieves the file path for the newly selected asset. |
| getPreviousAssetType() | String | O(1) | Retrieves the type identifier for the previously selected asset. May be null. |
| getPreviousAssetFilePath() | AssetPath | O(1) | Retrieves the file path for the previously selected asset. May be null. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly like a service. Instead, systems subscribe to it via an event bus. The standard pattern involves a listener method annotated to receive the event.

```java
// Example subscriber within an Asset Preview system
@Subscribe
public void onAssetSelected(AssetEditorSelectAssetEvent event) {
    // Unload the old asset to free resources
    if (event.getPreviousAssetFilePath() != null) {
        unloadAsset(event.getPreviousAssetFilePath());
    }

    // Load the new asset for display
    AssetPath newAsset = event.getAssetFilePath();
    if (newAsset != null) {
        loadAndDisplayAsset(newAsset);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Retaining References:** Do not store the event object in a field or collection after the listener method has completed. Extract the data you need and let the event be garbage collected. Storing the event can lead to memory leaks and stale state.
-   **Manual Instantiation:** Avoid creating an instance of this event simply to pass data between two tightly-coupled methods. Its design is for broadcast-based, decoupled communication. For direct method calls, pass the parameters directly.
-   **Ignoring Previous State:** Listeners that manage stateful resources (like textures or models) MUST handle the `previousAssetFilePath` to perform necessary cleanup. Ignoring it is a common source of resource leaks.

## Data Pipeline
The flow of information represented by this event is unidirectional, originating from user input and propagating through the system to all interested listeners.

> Flow:
> User Click in Asset Tree UI -> UI Event Listener -> **new AssetEditorSelectAssetEvent()** -> Editor Event Bus -> Subscribers (e.g., PreviewPane, PropertyEditor, WindowTitleUpdater)

