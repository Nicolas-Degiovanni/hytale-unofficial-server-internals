---
description: Architectural reference for PrefabEditorExitConfirmPage
---

# PrefabEditorExitConfirmPage

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.ui
**Type:** Transient

## Definition
```java
// Signature
public class PrefabEditorExitConfirmPage extends InteractiveCustomUIPage<PrefabEditorExitConfirmPage.PageData> {
```

## Architecture & Concepts
The PrefabEditorExitConfirmPage is a server-side component that represents a short-lived, interactive user interface dialog. Its primary function is to act as a gatekeeper, preventing accidental data loss when a user attempts to exit a prefab editing session with unsaved changes.

This class embodies the server-authoritative UI pattern prevalent in the Hytale engine. It does not contain rendering logic itself; instead, it functions as a controller that generates a series of commands for the client. These commands instruct the client to construct a specific UI from a template and bind user actions (like button clicks) to server-side event handlers.

It fits within the Builder Tools plugin as a critical state-management view, bridging the core session logic in PrefabEditSessionManager with direct player interaction. When a player triggers an exit condition, this page is instantiated to capture the player's intent—discard changes, cancel the exit, or proceed to a save workflow.

## Lifecycle & Ownership
- **Creation:** An instance is created exclusively when a player with unsaved modifications attempts to close the prefab editor. The PrefabEditSessionManager is the typical owner of this creation logic, gathering the necessary context (the player, the session, and a list of "dirty" prefabs) to construct the page.

- **Scope:** The object's lifetime is exceptionally brief and is tied directly to the visibility of the confirmation dialog on the client's screen. It exists only to handle a single, atomic user decision.

- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection immediately after the `handleDataEvent` method completes. In all logical paths, this method either closes the UI or replaces it with a new page (such as the PrefabEditorSaveSettingsPage). Once it is no longer the active page in the player's PageManager, no strong references remain.

## Internal State & Concurrency
- **State:** The internal state of this class is effectively **immutable**. All member fields are declared `final` and are injected during construction. The page's purpose is to act on this initial state, not to modify it over time. It serves as a read-only snapshot of the session's condition at the moment the exit was triggered.

- **Thread Safety:** This class is **not thread-safe** and is designed for single-threaded access within the context of a specific world's entity processing loop. All interactions, from creation to event handling, are expected to be serialized by the server's core engine. Unsynchronized access from other threads will lead to race conditions and unpredictable behavior.

## API Surface
The public contract is primarily defined by the InteractiveCustomUIPage inheritance, which are framework-level callbacks.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(...) | void | O(N) | **Framework Callback.** Populates UI commands to render the dialog. Complexity is linear to N, the number of dirty prefabs. |
| handleDataEvent(...) | void | O(1) | **Framework Callback.** Executes logic based on the player's choice. Triggers session exit, cancellation, or transitions to the save UI. |

## Integration Patterns

### Standard Usage
This class should never be instantiated directly by general-purpose game code. It is created and managed by the prefab editing session controller, which then passes it to the player's PageManager.

```java
// Correct usage within a session controller
// This code is conceptual and demonstrates the integration point.

List<PrefabEditingMetadata> dirtyPrefabs = session.findDirtyPrefabs();
if (!dirtyPrefabs.isEmpty()) {
    // The session manager creates the page with the required context
    PrefabEditorExitConfirmPage confirmPage = new PrefabEditorExitConfirmPage(
        playerRef,
        session,
        world,
        dirtyPrefabs
    );
    
    // The page is then displayed to the player
    player.getPageManager().openCustomPage(ref, store, confirmPage);
} else {
    // Exit directly if no changes
    session.exit();
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PrefabEditorExitConfirmPage()` in unrelated systems. The class requires specific, valid context from an active PrefabEditSession, and creating it without this context will result in runtime errors or broken behavior.

- **State Re-use:** Do not attempt to cache or re-use an instance of this page. It is a single-use object representing a specific state in time. If another confirmation is needed, a new instance must be created.

- **Manual Invocation:** Never call the `build` or `handleDataEvent` methods directly. They are part of a contract with the UI framework and are invoked automatically by the engine at the correct points in the lifecycle.

## Data Pipeline
The flow of data for this component is a complete round-trip, from server to client and back to the server.

> Flow:
> 1. **Server (Trigger):** A player action (e.g., running an exit command) is detected by the PrefabEditSessionManager.
> 2. **Server (Build):** A `PrefabEditorExitConfirmPage` is instantiated. The `build` method is invoked by the framework, generating UI commands.
> 3. **Network (Server -> Client):** The UI commands are serialized and sent to the client.
> 4. **Client:** The client's UI engine interprets the commands, renders the confirmation dialog, and binds button clicks to specific event data.
> 5. **Client (Interaction):** The player clicks a button (e.g., "Save and Exit").
> 6. **Network (Client -> Server):** The client sends a `CustomUIEvent` packet containing the bound event data (e.g., `Action: SaveAndExit`).
> 7. **Server (Deserialization):** The server receives the packet. The `PageData.CODEC` deserializes the payload into a `PageData` object.
> 8. **Server (Event Handling):** The framework invokes the `handleDataEvent` method on the **`PrefabEditorExitConfirmPage`** instance, passing the deserialized `PageData`.
> 9. **Server (Resolution):** The method executes the corresponding logic, such as transitioning the player to the `PrefabEditorSaveSettingsPage`.

