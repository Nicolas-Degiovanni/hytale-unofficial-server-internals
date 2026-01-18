---
description: Architectural reference for ImageImportPage
---

# ImageImportPage

**Package:** com.hypixel.hytale.builtin.buildertools.imageimport
**Type:** Transient

## Definition
```java
// Signature
public class ImageImportPage extends InteractiveCustomUIPage<ImageImportPage.PageData> {
```

## Architecture & Concepts
The ImageImportPage class is a server-side controller that powers the "Import Image" user interface for in-game builder tools. It is a stateful component responsible for managing the entire lifecycle of an image import operation for a single player, from initial UI rendering to the final creation of a block clipboard.

Architecturally, this class embodies the server-driven UI pattern. It does not render pixels directly; instead, it generates a series of commands via a UICommandBuilder that instruct the client on how to build and update the interface. All user interactions, such as typing in a file path or clicking a button, are sent from the client back to this specific page instance on the server for processing.

Its primary responsibilities include:
1.  **UI State Management:** Maintaining the state of all form inputs, such as the image path, maximum dimensions, and orientation.
2.  **File System Interaction:** Integrating with a ServerFileBrowser component to provide a secure and sandboxed way for players to select files from the server's designated *imports* directory.
3.  **Asynchronous Task Orchestration:** Delegating the computationally expensive task of image processing to a dedicated worker thread managed by the BuilderToolsPlugin. This is critical to prevent blocking the main server thread.
4.  **Player State Modification:** Upon successful import, it interfaces with the player's inventory and builder state to place the resulting block data into their clipboard and automatically switch them to the paste tool.

This class is a central hub, coordinating the UI system, the server file system, a background processing queue, and the core player game state.

### Lifecycle & Ownership
-   **Creation:** An instance of ImageImportPage is created by the server's PageManager when a player is directed to this UI, typically by executing a command like */imageimport*. A reference to the player, PlayerRef, is injected upon construction, binding this page instance exclusively to that player.
-   **Scope:** The object's lifetime is tied directly to the player's UI session. It persists as long as the player has the "Image Import" window open, holding their specific form data in memory.
-   **Destruction:** The instance is marked for garbage collection when the player closes the UI. This can happen explicitly (e.g., clicking a cancel button), implicitly (e.g., disconnecting), or upon successful completion of an import, which programmatically sets the player's current page to Page.None.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. It maintains the current values of all UI form fields (imagePath, maxDimension, orientation), UI visibility flags (showBrowser), and processing status (statusMessage, isProcessing). This state is modified exclusively in response to events from the associated player's client.

-   **Thread Safety:** **This class is not thread-safe and must only be accessed from the main server thread.** The public contract methods, build and handleDataEvent, are invoked by the server's core loop.

    To handle long-running operations, it employs an off-thread processing pattern. The performImport method does not process the image itself. Instead, it enqueues a task on the BuilderToolsPlugin's dedicated worker pool. The image loading, scaling, and color-matching logic occurs on a background thread. When the task is complete, the results (success or failure) are marshaled back to the main thread via a callback, which can then safely update the page's state and trigger a UI rebuild.

    **WARNING:** Any direct modification of this class's state from a background thread will lead to race conditions and server instability. All state changes must originate from the main server thread or be scheduled for execution on it.

## API Surface
The primary public contract is defined by its superclass, InteractiveCustomUIPage.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, commandBuilder, eventBuilder, store) | void | O(N) | Populates the command and event builders to construct or update the client-side UI. Complexity is proportional to the number of UI elements. |
| handleDataEvent(ref, store, data) | void | O(1) | The primary entry point for all client-initiated events. It routes incoming PageData to the appropriate state-change logic. |

## Integration Patterns

### Standard Usage
ImageImportPage is not meant to be instantiated directly. It is managed by the player's PageManager. To open this UI for a player, you set it as their active page.

```java
// Example from a server-side command handler
Player player = ...; // Get the player component
PlayerRef playerRef = ...; // Get the player reference

// The PageManager handles the lifecycle of the ImageImportPage instance
player.getPageManager().setPage(
    playerRef,
    store,
    new ImageImportPage(playerRef)
);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation and Management:** Do not create an instance with *new ImageImportPage()* and hold a reference to it. The server's UI framework is responsible for its lifecycle. Manual management will lead to memory leaks and a disconnect from the client UI state.
-   **Blocking Operations in handleDataEvent:** Never perform file I/O, image processing, or any other blocking operation directly within handleDataEvent. This will freeze the server tick. Always delegate heavy work to a worker thread, as demonstrated by the use of BuilderToolsPlugin.addToQueue.
-   **Cross-Player Access:** This page is intrinsically tied to a single player. Attempting to access or modify one player's page instance from the context of another player is an architectural violation and will cause unpredictable behavior.

## Data Pipeline
The flow of data through this component is bidirectional and involves multiple contexts: the client, the main server thread, and a background worker thread.

> **UI Rendering and Interaction Flow:**
> 1. Server Tick -> `build()` -> UICommandBuilder
> 2. UI Commands -> Network Packet -> Client UI Engine
> 3. Client Renders UI
> 4. User Input (e.g., Button Click) -> Client UI Event
> 5. UI Event -> Network Packet -> Server Network Layer
> 6. Server decodes packet into `PageData`
> 7. Server Tick -> `handleDataEvent(PageData)` -> **ImageImportPage** state is updated

> **Import Processing Flow:**
> 1. `handleDataEvent` triggers `performImport()` on Main Thread
> 2. `performImport()` -> Enqueues processing task via `BuilderToolsPlugin.addToQueue`
> 3. **Worker Thread** -> Reads file, processes image, maps colors, creates `BlockSelection`
> 4. **Worker Thread** -> Schedules callback for execution on Main Thread
> 5. **Main Thread** -> Executes callback -> `setError()` or updates player `builderState`
> 6. `rebuild()` is called -> Triggers a UI update via the standard rendering flow

