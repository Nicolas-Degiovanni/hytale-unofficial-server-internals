---
description: Architectural reference for DialogPage
---

# DialogPage

**Package:** com.hypixel.hytale.builtin.adventure.objectives
**Type:** Transient

## Definition
```java
// Signature
public class DialogPage extends InteractiveCustomUIPage<DialogPage.DialogPageEventData> {
```

## Architecture & Concepts
The DialogPage class is a server-side controller for a specific client-side user interface screen. It represents a concrete, interactive dialog box, typically used within the adventure mode objective system. This class follows a Model-View-Controller (MVC) pattern, where DialogPage acts as the Controller, the `Pages/DialogPage.ui` file is the View, and the `DialogOptions` asset is the Model.

Its primary responsibility is to bridge server-side game logic with the client's UI engine. When activated, it uses a `UICommandBuilder` to send a batch of instructions to the client, such as which UI layout to load and what text to display. It also registers event bindings via a `UIEventBuilder`, instructing the client to notify the server when specific UI elements, like a close button, are interacted with.

This class is a specialized implementation of the `InteractiveCustomUIPage` base class, which provides the core infrastructure for managing the lifecycle of custom UI pages tied to a specific player.

## Lifecycle & Ownership
- **Creation:** A DialogPage is instantiated on-demand by a higher-level game system, such as an objective or quest manager. This typically occurs when a player triggers an event that requires a dialog to be shown, for example, by interacting with an entity.
- **Scope:** The object's lifetime is ephemeral and strictly bound to the time the dialog is visible on the player's screen. It persists in memory on the server as part of the player's session state, managed by the `PageManager`.
- **Destruction:** The instance is marked for garbage collection as soon as the player's active page is changed. This is triggered within its own `handleDataEvent` method when the player clicks the close button, which calls `playerComponent.getPageManager().setPage(..., Page.None)`. The `PageManager` is then responsible for discarding its reference to the DialogPage instance.

## Internal State & Concurrency
- **State:** The internal state of DialogPage is minimal and effectively immutable after construction. It holds a final reference to `dialogOptions`, which contains static configuration data loaded from game assets. It does not cache or manage any dynamic or mutable state.
- **Thread Safety:** This class is **not thread-safe** and must only be accessed from the server's main world-tick thread for the corresponding player. The Hytale server architecture guarantees that all player-related UI events and build commands are processed sequentially within this single-threaded context, eliminating the need for explicit locking.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DialogPage(playerRef, dialogOptions) | constructor | O(1) | Creates a new dialog page instance for a specific player with the given dialog content. |
| build(ref, commandBuilder, eventBuilder, store) | void | O(1) | Populates the command and event builders with instructions to render the dialog on the client. |
| handleDataEvent(ref, store, data) | void | O(1) | Handles the incoming event from the client, typically to close the dialog page. |

## Integration Patterns

### Standard Usage
A DialogPage should be instantiated and immediately passed to the player's `PageManager` to be set as the active page. The engine handles the subsequent calls to `build` and the routing of UI events to `handleDataEvent`.

```java
// Executed within a server-side system (e.g., an objective task)
Player playerComponent = store.getComponent(playerRef, Player.getComponentType());
UseEntityObjectiveTaskAsset.DialogOptions options = ...; // Load from asset

// 1. Create the page instance
DialogPage dialog = new DialogPage(playerRef, options);

// 2. Set it as the player's active page
playerComponent.getPageManager().setPage(playerRef, store, dialog);
```

### Anti-Patterns (Do NOT do this)
- **Reusing Instances:** A DialogPage instance is single-use. Do not attempt to cache and re-display a closed DialogPage. A new instance must be created for every dialog interaction to ensure a clean state.
- **Manual Event Invocation:** Never call `handleDataEvent` directly from your own code. This method is exclusively for the server's UI event dispatch system. Manually invoking it will bypass the network protocol and lead to state desynchronization between the client and server.
- **Stateful Logic:** Avoid adding mutable state to subclasses of DialogPage. The page's lifecycle is short and unpredictable from an external perspective. All required data should be passed in via the constructor.

## Data Pipeline
The flow of data for a DialogPage involves a full round-trip from server to client and back.

> **Flow (Display):**
> Server Game Logic -> `new DialogPage(...)` -> `PageManager.setPage(...)` -> Engine calls `DialogPage.build()` -> UI Commands Serialized -> **Network Packet to Client**
>
> **Flow (Interaction):**
> Client Renders UI -> Player Clicks Button -> **Network Packet to Server** -> UI Event Deserialized -> Engine routes to `DialogPage.handleDataEvent()` -> `PageManager.setPage(..., Page.None)` -> UI Closed

