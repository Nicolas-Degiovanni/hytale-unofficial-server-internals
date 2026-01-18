---
description: Architectural reference for WarpListPage
---

# WarpListPage

**Package:** com.hypixel.hytale.builtin.teleport
**Type:** Transient

## Definition
```java
// Signature
public class WarpListPage extends InteractiveCustomUIPage<WarpListPage.WarpListPageEventData> {
```

## Architecture & Concepts
The WarpListPage class is a server-side controller for a player-facing custom user interface. It does not represent the UI visually, but rather orchestrates its construction, state management, and event handling. It is a concrete implementation of the `InteractiveCustomUIPage` base class, which firmly places it within the server's authoritative UI framework.

Its primary responsibility is to render a dynamic, searchable list of available `Warp` locations to a player and process their selection. This is achieved by generating a series of UI commands via the `UICommandBuilder` and defining event bindings with the `UIEventBuilder`. These builders produce data structures that are serialized and sent to the client, which then renders the interface accordingly.

Client interactions, such as clicking a warp or typing in a search field, are sent back to the server as serialized events. The `WarpListPageEventData` inner class and its associated `CODEC` are responsible for deserializing this incoming data. The `handleDataEvent` method then acts upon this data, either by invoking the final selection callback or by rebuilding and sending a UI update to the client to reflect the new search results.

This class is a prime example of a server-driven UI pattern, where the client acts as a thin rendering terminal and all logic and state are managed authoritatively by the server.

### Lifecycle & Ownership
-   **Creation:** An instance is created on-demand by a higher-level game system, typically a command handler (e.g., a `/warps` command). The creator must supply a reference to the target player, a map of available warps, and a `Consumer` callback function. This callback is the critical link for returning the player's selection to the calling system.
-   **Scope:** The object's lifetime is tied directly to the time the UI is visible to the player. It is held as the active page within the target player's `PageManager` component. It persists as long as the player is interacting with the warp list.
-   **Destruction:** The instance becomes eligible for garbage collection as soon as it is no longer the active page. This happens under two conditions:
    1.  The player selects a warp, causing `handleDataEvent` to call `playerComponent.getPageManager().setPage(ref, store, Page.None)`.
    2.  The player closes the UI or another system replaces the page.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. It maintains the player's current `searchQuery` as internal state, which is modified when the player types in the search input field. The list of warps and the callback are provided at construction and are treated as immutable for the object's lifetime.
-   **Thread Safety:** This class is **not thread-safe** and must only be accessed from the main server thread. It is designed to operate within the server's single-threaded, tick-based game loop. All methods, including the constructor, `build`, and `handleDataEvent`, are expected to be called synchronously on the same thread that manages the player's entity. Unsynchronized access from other threads will lead to race conditions and unpredictable UI state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WarpListPage(playerRef, warps, callback) | constructor | O(1) | Instantiates a new UI page controller for a specific player interaction. |
| build(ref, commandBuilder, eventBuilder, store) | void | O(N) | Populates the builders for the initial construction of the UI. Complexity is linear to the number of warps. |
| handleDataEvent(ref, store, eventData) | void | O(1) or O(N) | Processes an incoming event from the client. O(1) for a warp selection; O(N) for a search query update, as it triggers a full list rebuild. |

## Integration Patterns

### Standard Usage
The class is intended to be instantiated and presented to a player by a system that needs to prompt for a warp location. The system provides a callback to define what happens after the selection is made.

```java
// Example from a hypothetical command execution context
PlayerRef playerRef = ...;
Map<String, Warp> availableWarps = warpManager.getWarps();

// Define the action to take upon selection
Consumer<String> onWarpSelected = (warpName) -> {
    Warp targetWarp = availableWarps.get(warpName);
    teleportSystem.teleportPlayer(playerRef, targetWarp.getLocation());
};

// Create and show the page
WarpListPage warpPage = new WarpListPage(playerRef, availableWarps, onWarpSelected);
player.getPageManager().setPage(playerRef, store, warpPage);
```

### Anti-Patterns (Do NOT do this)
-   **Instance Re-use:** Do not cache and re-use a `WarpListPage` instance. Its internal state (like `searchQuery`) is specific to a single interaction. A new instance must be created each time the UI is shown.
-   **External State Mutation:** Do not modify the `warps` map passed into the constructor after the `WarpListPage` has been created. The class does not create a defensive copy and assumes the collection is static for its lifetime.
-   **Asynchronous Invocation:** Never call `handleDataEvent` or other methods from a separate thread. All interactions must be synchronized with the server's main tick.

## Data Pipeline
The flow of data for this component involves a round-trip from server to client and back.

> **Initial Display Flow:**
> Server Logic (e.g., Command) → `new WarpListPage()` → `PageManager.setPage()` → `WarpListPage.build()` → `UICommandBuilder` → Network Packet → Client UI Engine Renders Page

> **Search Update Flow:**
> Client Keyboard Input → Network Packet (containing `WarpListPageEventData`) → Server Network Layer → **`WarpListPage.handleDataEvent()`** → `buildWarpList()` → `sendUpdate()` → Network Packet → Client UI Engine Re-renders List

> **Selection Flow:**
> Client Button Click → Network Packet (containing `WarpListPageEventData`) → Server Network Layer → **`WarpListPage.handleDataEvent()`** → `callback.accept()` → Server Logic (e.g., Teleport)

