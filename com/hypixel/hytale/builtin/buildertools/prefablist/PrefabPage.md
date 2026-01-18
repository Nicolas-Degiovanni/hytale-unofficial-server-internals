---
description: Architectural reference for PrefabPage
---

# PrefabPage

**Package:** com.hypixel.hytale.builtin.buildertools.prefablist
**Type:** Transient

## Definition
```java
// Signature
public class PrefabPage extends InteractiveCustomUIPage<FileBrowserEventData> {
```

## Architecture & Concepts

The PrefabPage class is the server-side controller responsible for managing the "Prefab Selection" user interface. It is a specialized implementation of InteractiveCustomUIPage, designed to operate within the Builder Tools plugin ecosystem. It is not a UI component itself, but rather the backend logic that powers the client-side UI defined in `Pages/PrefabListPage.ui`.

Its primary architectural function is to provide a unified browsing experience for two distinct sources of prefabs:
1.  **Server Prefabs:** Physical `.prefab.json` files stored on the server's file system, typically for world-specific or server-specific structures.
2.  **Asset Prefabs:** Virtual prefab files packaged within the game's core assets, representing default or built-in structures.

To achieve this, PrefabPage composes a generic **ServerFileBrowser** utility for handling the standard file system navigation (Server Prefabs). For Asset Prefabs, it implements a parallel navigation logic using a custom **AssetPrefabFileProvider**, effectively creating a virtual file system layer. The class maintains state to seamlessly switch between these two modes based on user selection in the UI's root selector.

The ultimate purpose of this class is to translate a player's UI selection of a prefab file into a concrete gameplay action: loading the selected prefab data into the player's **Paste Tool** and automatically switching to it.

### Lifecycle & Ownership
-   **Creation:** A new PrefabPage instance is created by the BuilderToolsPlugin system whenever a player initiates an action to open the prefab browser. It is instantiated with a direct reference to the player (PlayerRef) and the current builder state.
-   **Scope:** The object's lifetime is strictly bound to the player's UI session. It persists only while the prefab selection interface is visible to the player.
-   **Destruction:** The instance is discarded and becomes eligible for garbage collection when the player closes the UI. This occurs either by explicitly dismissing the page or by successfully selecting a prefab, which triggers a call to `playerComponent.getPageManager().setPage(ref, store, Page.None)`.

## Internal State & Concurrency
-   **State:** PrefabPage is a stateful class. Its state is mutable and represents the current view of the UI for a single player. It tracks the active file system root (Assets vs. Server), the current navigation path within the virtual asset system (assetsCurrentDir), and delegates file system state (search queries, server paths) to its internal ServerFileBrowser instance.
-   **Thread Safety:** This class is **not thread-safe** and must not be accessed from outside the owner player's entity processing thread. All its methods assume single-threaded execution within the server's game loop tick. Any concurrent modification would corrupt the UI state and lead to undefined behavior.

## API Surface

The public contract is defined by its inheritance from InteractiveCustomUIPage. These methods are callbacks invoked by the server's UI page management system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(...) | void | O(N) | Invoked once when the page is first opened. Constructs and sends the initial UI commands to the client. Complexity is proportional to the number of files (N) in the initial directory. |
| handleDataEvent(...) | void | O(1) | The primary event handler. Invoked when the client sends interaction data. Dispatches logic based on the event type (root change, search, file selection). |

## Integration Patterns

### Standard Usage

PrefabPage is not intended for direct manual management. It is designed to be instantiated and managed by the player's PageManager, typically triggered by a plugin command.

```java
// Correct instantiation within the plugin framework
Player player = ...;
PlayerRef playerRef = ...;
BuilderToolsPlugin.BuilderState state = ...;

// The PageManager takes ownership of the new page instance
player.getPageManager().setPage(
    ref,
    store,
    new PrefabPage(playerRef, defaultPath, state)
);
```

### Anti-Patterns (Do NOT do this)
-   **State Caching:** Do not hold a reference to a PrefabPage instance after it has been closed. Each time the UI is opened, a new instance must be created to ensure a clean state.
-   **External State Mutation:** Do not attempt to modify the internal ServerFileBrowser or other state fields directly. All state transitions must be driven through the `handleDataEvent` method by simulating UI events.
-   **Cross-Player Usage:** An instance of PrefabPage is inextricably linked to a single PlayerRef. Attempting to use it for multiple players will cause severe state corruption and network packet misdirection.

## Data Pipeline

The flow of data from user interaction to game state change follows a clear client-server-client path.

> Flow:
> Client UI Click (e.g., on a file name) -> Client sends `CustomUIEvent` packet with `FileBrowserEventData` -> Server Network Layer -> Player's PageManager -> **PrefabPage.handleDataEvent** -> `handlePrefabSelection` -> `PrefabStore.get()` -> `BuilderToolsPlugin.addToQueue()` -> Player Inventory is modified -> Server sends `SetActiveSlot` packet -> Client hotbar updates.

