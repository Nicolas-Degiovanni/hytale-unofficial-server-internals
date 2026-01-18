---
description: Architectural reference for AssetEditorAssetCreatedEvent
---

# AssetEditorAssetCreatedEvent

**Package:** com.hypixel.hytale.builtin.asseteditor.event
**Type:** Transient

## Definition
```java
// Signature
public class AssetEditorAssetCreatedEvent extends EditorClientEvent<String> {
```

## Architecture & Concepts
The AssetEditorAssetCreatedEvent is an immutable data-transfer object that represents a discrete occurrence within the Hytale Asset Editor: the successful creation of a new asset by the user.

This class is a fundamental component of the editor's event-driven architecture. It serves to decouple the user interface, which originates the creation request, from the various backend systems that must react to it. Instead of a UI controller directly invoking services for file I/O, UI tree updates, and logging, it simply instantiates and dispatches this event. This promotes a clean separation of concerns, allowing new listeners to be added without modifying the event's source.

As a subclass of EditorClientEvent, it operates within the context of a specific EditorClient session, ensuring that events are routed and handled appropriately for the active editor instance.

## Lifecycle & Ownership
- **Creation:** Instantiated by a UI controller or a high-level editor service immediately after a user confirms the creation of a new asset. The constructor is populated with all necessary context, including the asset's type, path, raw byte data, and the UI element that triggered the action.
- **Scope:** Extremely short-lived. The event object exists only for the duration of its dispatch through the event bus. It is a "fire-and-forget" message.
- **Destruction:** Becomes eligible for garbage collection as soon as all registered event listeners have completed processing it. No system should maintain a long-term reference to an instance of this event.

## Internal State & Concurrency
- **State:** **Immutable**. All member fields are declared as final and are initialized exclusively through the constructor. This design guarantees that the event's data cannot be altered after its creation, which is critical for ensuring predictable behavior across multiple, independent listeners.
- **Thread Safety:** Inherently **Thread-Safe**. Due to its immutability, an instance of AssetEditorAssetCreatedEvent can be safely passed across thread boundaries without the need for locks or other synchronization primitives. This is essential in a multi-threaded engine where UI events may be processed by worker threads.

## API Surface
The public API consists solely of accessor methods to retrieve the event's payload.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetType() | String | O(1) | Returns the category of the created asset, such as "model" or "texture". |
| getAssetPath() | Path | O(1) | Returns the intended, fully-qualified file system path for the new asset. |
| getData() | byte[] | O(1) | Returns the raw binary content of the asset. **Warning:** This may hold a large amount of data in memory. |
| getButtonId() | String | O(1) | Returns the unique identifier of the UI button that initiated the creation, allowing for contextual feedback. |

## Integration Patterns

### Standard Usage
This event is intended to be published to a central event bus or dispatcher. Subscribing systems listen for this specific event type and execute their logic based on its payload.

```java
// A listener system reacting to the event
public void onAssetCreated(AssetEditorAssetCreatedEvent event) {
    Path destination = event.getAssetPath();
    byte[] content = event.getData();

    // Persist the new asset to the disk
    fileSystemService.writeBytes(destination, content);

    // Update the asset browser UI to show the new file
    assetBrowser.addNode(destination);
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Retention:** Do not store references to this event in caches or long-lived collections. The `data` byte array can be large, and retaining the event object will cause a memory leak. Process it and discard it.
- **State Modification:** Do not use reflection or other means to attempt to modify the event's internal state. Its immutability is a core design contract.
- **Synchronous Dependency:** Do not design systems that block and wait for a response after dispatching this event. It is part of an asynchronous, fire-and-forget messaging pattern.

## Data Pipeline
The event acts as a message carrying a payload from the user interface to backend services.

> Flow:
> User Action in Editor UI -> UI Controller Logic -> **AssetEditorAssetCreatedEvent Instantiation** -> Event Bus Dispatch -> Subscribed Listeners (e.g., FileSystemWriter, AssetBrowserUpdater) -> Asset persisted to disk & UI refreshed

