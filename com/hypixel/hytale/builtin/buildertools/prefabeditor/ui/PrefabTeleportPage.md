---
description: Architectural reference for PrefabTeleportPage
---

# PrefabTeleportPage

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.ui
**Type:** Transient

## Definition
```java
// Signature
public class PrefabTeleportPage extends InteractiveCustomUIPage<PrefabTeleportPage.PageData> {
```

## Architecture & Concepts
The PrefabTeleportPage class is a server-side controller that manages a specific client-side user interface for the in-game builder tools. It is not a visual component itself; rather, it generates a declarative definition of a UI which is then sent to the client for rendering. This class embodies a server-authoritative UI pattern, where the server dictates the structure and content of the UI and processes all interactions.

Its primary role is to bridge the data model, represented by the PrefabEditSession, with the player's interactive UI. It is responsible for:
1.  Querying the available prefabs from the current editing session.
2.  Building a dynamic, scrollable list of these prefabs for the player.
3.  Implementing a real-time search and filter functionality over the prefab list.
4.  Handling the user's selection to trigger a teleport action.

This component operates entirely within the server's main game loop and is tightly coupled to the lifecycle of a specific player's UI state.

### Lifecycle & Ownership
-   **Creation:** An instance of PrefabTeleportPage is created on-demand when a player performs an action that requires teleporting to a prefab, such as interacting with a specific builder tool menu. It is instantiated and assigned to the player via the player's PageManager.
-   **Scope:** The object's lifetime is ephemeral, existing only as long as the player has this specific UI page open. It is scoped to a single player and a single PrefabEditSession.
-   **Destruction:** The instance is marked for garbage collection when the player's active page is changed. This occurs either when the player successfully teleports (which programmatically sets the page to None) or when the player manually dismisses the UI.

## Internal State & Concurrency
-   **State:** This class is stateful. It maintains a mutable internal state, specifically the **searchQuery** string, which is updated based on user input from the client. It also holds an immutable reference to the **PrefabEditSession** for the duration of its life. The list of prefabs displayed is not cached; it is rebuilt from the PrefabEditSession on every state change.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be accessed exclusively from the server's main world thread. All interactions, from the initial build to event handling, are synchronized by the game loop. Unsynchronized access from other threads will result in race conditions, UI corruption, and unpredictable server behavior.

## API Surface
The public contract is primarily defined by framework-level overrides rather than a direct-call API.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, commandBuilder, eventBuilder, store) | void | O(N log N) | Framework callback to construct the initial UI state. Complexity is dominated by sorting N prefabs. |
| handleDataEvent(ref, store, data) | void | O(N log N) | Framework callback to process UI events from the client. Search events trigger a full list rebuild and sort. Teleport events are O(1) plus world data access time. |

## Integration Patterns

### Standard Usage
This page should only be managed through a player's PageManager. The system responsible for builder tools UI will create an instance and set it as the player's active page.

```java
// Example of how another system would open this page for a player
PrefabEditSession session = ... // Obtain the current session
PlayerRef playerRef = ... // Get the target player's reference

// The PageManager takes ownership and manages the lifecycle
player.getPageManager().setPage(ref, store, new PrefabTeleportPage(playerRef, session));
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation without Management:** Do not create an instance with *new* and attempt to use it without passing it to a PageManager. A standalone instance is inert; it will not be displayed to the client or receive any events.
-   **External State Mutation:** Do not attempt to modify the internal **searchQuery** field from outside the class. All state changes must be driven by client UI events processed through the **handleDataEvent** method to ensure a consistent and predictable UI flow.
-   **Cross-Thread Access:** Never access an instance of this class from a thread other than the main server thread for the player's world.

## Data Pipeline
The class facilitates two primary data flows: a reactive search loop and a one-shot teleport action.

**Search & Filter Flow:**
> Client UI (User types in search box) → Network Event Packet → Server Event Dispatcher → **PrefabTeleportPage.handleDataEvent** → Internal *searchQuery* state is updated → **PrefabTeleportPage.buildPrefabList** → UICommandBuilder generates update → Server sends UI update packet → Client UI (List is re-rendered)

**Teleport Action Flow:**
> Client UI (User clicks a prefab button) → Network Event Packet → Server Event Dispatcher → **PrefabTeleportPage.handleDataEvent** → Prefab metadata is retrieved from PrefabEditSession → World data is queried for a safe Y-coordinate → A **Teleport** component is added to the Player entity → The Teleportation System processes the component in a subsequent tick → Player's position is updated → The UI is closed by setting the active page to None.

