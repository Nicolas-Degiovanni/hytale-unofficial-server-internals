---
description: Architectural reference for PrefabEditorLoadSettingsPage
---

# PrefabEditorLoadSettingsPage

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.ui
**Type:** Transient

## Definition
```java
// Signature
public class PrefabEditorLoadSettingsPage extends InteractiveCustomUIPage<PrefabEditorLoadSettingsPage.PageData> {
```

## Architecture & Concepts
The PrefabEditorLoadSettingsPage class is a server-side controller that manages the user interface for loading prefabs into a dedicated editing session. It does not render pixels directly; instead, it defines the logic, state, and event bindings for a corresponding UI layout file (likely `Pages/PrefabEditorSettings.ui`) that is rendered on the client.

This class acts as the primary bridge between a player's UI interaction and the server's powerful prefab processing systems. Its core responsibility is to gather a complex set of configuration options from the player—such as file paths, world generation parameters, and rendering settings—and orchestrate the asynchronous creation of a prefab editing world via the PrefabEditSessionManager.

Architecturally, it follows a server-authoritative UI model:
1.  **Model:** The internal fields of the class (e.g., `isLoading`, `browserCurrent`) and the nested PageData class represent the state.
2.  **View:** A client-side UI file defines the visual layout and components.
3.  **Controller:** This class implements the controller logic, responding to client-sent events and issuing commands to update the client's view.

This separation allows complex, stateful operations like file browsing and asynchronous world creation to be managed securely on the server while providing a responsive user experience on the client.

### Lifecycle & Ownership
- **Creation:** An instance of PrefabEditorLoadSettingsPage is created by the server's PageManager when a player is directed to this specific UI. This is typically triggered by a server command, such as `/editprefab load`. It is **not** instantiated directly by developers. Each instance is uniquely tied to a single player via the PlayerRef passed into its constructor.

- **Scope:** The object's lifetime is bound to the player's interaction with this specific UI screen. It persists as long as the "Load Prefab" settings page is visible to the player.

- **Destruction:** The instance is marked for garbage collection when the player navigates away from the page, either by successfully loading a prefab, cancelling the operation, or being moved to another page by the server (e.g., `player.getPageManager().setPage(ref, store, Page.None)`).

## Internal State & Concurrency
- **State:** This class is highly stateful and mutable. It maintains the state of the UI form, the file browser's current path, the list of selected items, and the status of an active loading operation. Key state fields include `isLoading`, `loadingCancelled`, `currentLoadingState`, and `browserCurrent`.

- **Thread Safety:** This class is **not thread-safe** and its methods should only be invoked from the main server thread. However, it is designed to operate within an asynchronous environment. The prefab loading process is offloaded to a worker thread via a CompletableFuture returned by `PrefabEditSessionManager.loadPrefabAndCreateEditSession`.

    Callbacks such as `onLoadingProgress` and `onLoadingFailed` are executed by the CompletableFuture's thread pool. These methods safely construct UICommandBuilder objects and dispatch them via the `sendUpdate` method, which is assumed to handle the necessary thread synchronization to safely update the player's client. The use of `volatile` for fields like `isLoading` and `loadingCancelled` ensures visibility of state changes across threads, preventing race conditions during the cancellation process.

## API Surface
The public contract is defined by the InteractiveCustomUIPage base class. Developers do not call these methods directly; they are invoked by the server's UI framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, commandBuilder, eventBuilder, store) | void | O(N) | Called once upon page creation. Populates the UI with initial data (e.g., dropdowns for environments, axes) and binds all client-side UI events (e.g., button clicks) to server-side Action enums. Complexity is proportional to the number of UI elements being configured. |
| handleDataEvent(ref, store, data) | void | O(1) | The primary event handler. Called by the framework when a player interacts with a bound UI element. It uses a switch statement on the deserialized PageData to route control flow to the appropriate logic for loading, saving, browsing, or cancelling. |

## Integration Patterns

### Standard Usage
This class is not used directly. It is integrated into the server's command and UI page management system. A command handler is the typical entry point for a player to access this page.

```java
// In a hypothetical command execution handler
// This is the correct way to show the UI to a player.
// The framework handles the instantiation and lifecycle of the page.

Player player = ...;
EntityStore store = ...;
Ref<EntityStore> playerRef = ...;

// The PageManager will create a new PrefabEditorLoadSettingsPage
// instance for this specific player.
player.getPageManager().setPage(playerRef, store, new PrefabEditorLoadSettingsPage(player.getPlayerRef()));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PrefabEditorLoadSettingsPage()` for any purpose other than passing it to the `PageManager`. The framework must own and manage its lifecycle.

- **External State Mutation:** Do not attempt to modify the internal state of this page from another thread or system. Its state is tightly coupled to the UI interaction loop.

- **Blocking Operations:** Do not perform long-running or blocking operations within the `handleDataEvent` method. This will freeze the player's UI and potentially the server thread. All heavy lifting, like loading prefabs, is correctly delegated to asynchronous systems like the `PrefabEditSessionManager`.

## Data Pipeline
The flow of data for the primary "Load" action demonstrates the client-server interaction model.

> Flow:
> 1. **Client Interaction:** Player configures settings and clicks the "Load" button.
> 2. **Client-Side Event:** The client sends a `CustomUIEvent` packet containing the bound Action (`Load`) and all form data.
> 3. **Server Deserialization:** The server's network layer receives the packet and the UI framework deserializes its payload into a `PrefabEditorLoadSettingsPage.PageData` object.
> 4. **Event Handling:** The framework invokes `handleDataEvent` on the player's specific `PrefabEditorLoadSettingsPage` instance.
> 5. **State Transition:** The handler sets `isLoading` to true and switches the client's view to the loading screen via a `UICommandBuilder`.
> 6. **Asynchronous Task:** The handler calls `PrefabEditSessionManager.loadPrefabAndCreateEditSession`, which starts the world creation process on a background thread.
> 7. **Progress Updates:** The session manager periodically invokes the `onLoadingProgress` callback. This callback builds and sends new `UICommandBuilder` updates to the client to advance the progress bar and update status text.
> 8. **Completion:** Upon completion (success or failure), the `CompletableFuture` invokes the final callback, which either transitions the player to the new world or displays a final error message on the UI.

