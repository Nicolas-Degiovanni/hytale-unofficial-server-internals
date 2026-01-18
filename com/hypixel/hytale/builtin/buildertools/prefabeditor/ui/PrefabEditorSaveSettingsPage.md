---
description: Architectural reference for PrefabEditorSaveSettingsPage
---

# PrefabEditorSaveSettingsPage

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.ui
**Type:** Transient

## Definition
```java
// Signature
public class PrefabEditorSaveSettingsPage extends InteractiveCustomUIPage<PrefabEditorSaveSettingsPage.PageData> {
```

## Architecture & Concepts
The PrefabEditorSaveSettingsPage class is a server-side Controller that manages the state and logic for the client-side "Save Prefab" user interface. It is a critical component of the in-game builder tools, acting as the bridge between player input and the server's file system for prefab persistence.

This class follows a stateful, event-driven model. It does not render UI directly; instead, it sends declarative commands to the client using a UICommandBuilder. When a player interacts with the UI (e.g., clicks a button, types in a search box), the client sends a corresponding event packet to the server. The server decodes this into a PageData object and invokes the handleDataEvent method, which serves as the central dispatcher for all user actions.

Its primary responsibilities include:
-   Initializing the UI with default values and a list of loaded prefabs.
-   Handling user interactions such as selecting prefabs, toggling save options (e.g., include entities, overwrite), and searching.
-   Orchestrating the asynchronous prefab saving process via the PrefabSaver utility.
-   Providing real-time feedback to the player, such as progress bars, status updates, and error messages, by sending incremental UI updates.

This class encapsulates complex UI state, including the player's search query and the set of selected prefabs, ensuring that all logic related to this specific UI screen is co-located and managed coherently.

### Lifecycle & Ownership
-   **Creation:** An instance is created and assigned to a player when they initiate the "save" action within the prefab editor. The server's PageManager is responsible for its instantiation and presentation to the client.
-   **Scope:** The object's lifetime is tied directly to the player's UI session. It persists only as long as the "Save Prefab Settings" dialog is visible on the player's screen.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the player closes the page, either by successfully saving, canceling the operation, or disconnecting from the server. The PageManager manages this lifecycle implicitly.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. It maintains the current state of the UI form, including the `isSaving` flag, the `browserSearchQuery` string, and the `selectedPrefabUuids` set. This state is modified in response to user events received in `handleDataEvent`.

-   **Thread Safety:** This class is **not thread-safe** and is designed to be accessed only from the main server thread (the world tick thread). However, it initiates asynchronous operations for file I/O to avoid blocking the server. The `isSaving` field is marked as **volatile** to ensure that its value is immediately visible to all threads. This prevents race conditions where a player might attempt to start a new save operation while a previous one is still in progress on a background thread. All UI updates originating from background threads (via `CompletableFuture` callbacks) must be marshaled back to the main thread before being sent to the client.

## API Surface
The public API is primarily designed for interaction with the server's UI framework, not for direct developer invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, commandBuilder, eventBuilder, store) | void | O(N) | Called once by the PageManager to perform initial setup of the UI. Binds all static event listeners and sets the initial state of UI components. N is the number of UI elements. |
| handleDataEvent(ref, store, data) | void | O(M) | The primary event handler. Dispatches actions based on the incoming PageData. Complexity varies; simple actions are O(1), while saving is O(M) where M is the number of prefabs to save. |

## Integration Patterns

### Standard Usage
This class is not meant to be used directly. The engine's `PageManager` instantiates and manages it as part of the player's UI state. The only valid way to display this page is through the page management system.

```java
// Correctly displaying the page for a player
Player player = ...;
PrefabEditSession session = ...;
player.getPageManager().setPage(ref, store, new PrefabEditorSaveSettingsPage(player.getRef(), session));
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not create an instance of PrefabEditorSaveSettingsPage without immediately passing it to a PageManager. An orphaned instance has no connection to a player and cannot receive events or send updates.
-   **External State Mutation:** Never modify the internal state of this class (e.g., `isSaving`, `selectedPrefabUuids`) from another system. All state changes must be driven by events processed through `handleDataEvent` to ensure UI consistency.
-   **Synchronous Saving:** The save operation uses `CompletableFuture` to prevent freezing the server. Do not attempt to block the main thread by calling `.join()` or `.get()` on the futures returned by PrefabSaver from within the event handler.

## Data Pipeline
The flow of data for a typical save operation demonstrates the class's role as a central coordinator between the client, the game state, and the file system.

> Flow:
> 1.  Client UI Interaction (Player clicks "Save")
> 2.  -> Client sends CustomUIEvent packet
> 3.  -> Server Network Layer decodes packet into `PageData`
> 4.  -> `PageManager` routes `PageData` to `PrefabEditorSaveSettingsPage.handleDataEvent`
> 5.  -> **PrefabEditorSaveSettingsPage** validates input, sets `isSaving` state, and sends "Saving..." UI update to client
> 6.  -> **PrefabEditorSaveSettingsPage** invokes `PrefabSaver.savePrefab` for each selected prefab, creating multiple `CompletableFuture` instances
> 7.  -> I/O Worker Thread performs file writing
> 8.  -> `CompletableFuture` callback (on worker thread) triggers UI progress update
> 9.  -> **PrefabEditorSaveSettingsPage** sends `UICommandBuilder` update to client
> 10. -> Client UI renders the progress bar update
> 11. -> After all futures complete, a final success/failure message is sent and the page is closed.

