---
description: Architectural reference for SetNameRespawnPointPage
---

# SetNameRespawnPointPage

**Package:** com.hypixel.hytale.builtin.beds.respawn
**Type:** Transient

## Definition
```java
// Signature
public class SetNameRespawnPointPage extends RespawnPointPage {
```

## Architecture & Concepts
The SetNameRespawnPointPage is a server-side controller that manages the user interface for naming or renaming a player's respawn point. It is a concrete implementation of the abstract Page concept, which represents a single, stateful UI screen presented to a player.

This class acts as a bridge between a player's direct interaction with a RespawnBlock in the world (such as a bed) and the persistence of that respawn location in the player's data. It follows a controller pattern:
1.  The **build** method acts as the view setup, sending commands to the client to render the UI, pre-populate the name input field, and bind server-side events to UI buttons.
2.  The **handleDataEvent** method acts as the event handler, processing data sent back from the client when the player interacts with the UI, such as clicking the "Set" or "Cancel" buttons.

It is a critical component of the respawn mechanic, providing the necessary player-facing interface to manage custom-named spawn locations.

## Lifecycle & Ownership
-   **Creation:** An instance is created by the server's block interaction system when a player interacts with a RespawnBlock in a way that triggers the naming UI. The context of the interaction (player, block position, block type) is injected via the constructor.
-   **Scope:** This object is extremely short-lived. Its lifecycle is tied directly to the visibility of the "Name Respawn Point" UI on the player's client. It exists only for the duration of that specific UI session.
-   **Destruction:** The object is marked for garbage collection as soon as the UI is closed. This happens explicitly within the **handleDataEvent** method when the player confirms or cancels, which calls `playerComponent.getPageManager().setPage(ref, store, Page.None)`, effectively replacing this page and releasing all references to it.

## Internal State & Concurrency
-   **State:** The class is stateful, but its state is effectively immutable after construction. The `respawnBlockPosition` and `respawnBlock` fields are final and represent the context of the interaction that created the page. It does not cache or modify this state during its lifetime.
-   **Thread Safety:** This class is **not thread-safe** and must not be accessed from multiple threads. It is designed to be owned and operated on exclusively by the server's main thread within the context of a specific player's updates. All method calls are expected to be serialized by the game loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(Ref, UICommandBuilder, UIEventBuilder, Store) | void | O(N) | Configures the client-side UI. Populates the input field with an existing name if one is found. Complexity is O(N) where N is the number of the player's existing respawn points. |
| handleDataEvent(Ref, Store, RespawnPointEventData) | void | O(1) | Callback executed when the client sends UI data. It either sets the new respawn point name or closes the page. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most developers. It is instantiated and managed internally by the core game systems that handle block interactions. The system pushes an instance of this page to the player's PageManager, which then drives its lifecycle.

```java
// Conceptual example of how the system uses this class
// This code would exist within a block interaction handler.

PlayerRef playerRef = ...;
Vector3i blockPos = ...;
RespawnBlock respawnBlock = ...;

// Create the page with the interaction context
SetNameRespawnPointPage page = new SetNameRespawnPointPage(playerRef, InteractionType.USE, blockPos, respawnBlock);

// Assign the page to the player, causing the UI to appear on their client
player.getPageManager().setPage(ref, store, page);
```

### Anti-Patterns (Do NOT do this)
-   **State Re-use:** Do not hold a reference to this object after the UI has been closed. It is a transient object and its context becomes invalid once the player's PageManager moves to a different page.
-   **Manual Event Invocation:** Do not call **handleDataEvent** directly. It is designed to be a callback invoked by the server's UI event processing pipeline in response to a network packet from the client.

## Data Pipeline
The flow of data for this component involves two distinct phases: UI construction and event handling.

> **UI Construction Flow:**
> Player interacts with RespawnBlock -> Server Block Interaction Handler -> **new SetNameRespawnPointPage()** -> Player.PageManager.setPage() -> **build()** -> UICommandBuilder sends commands to client -> Client renders UI

> **Event Handling Flow:**
> Player clicks button in UI -> Client sends CustomUIEvent packet -> Server Network Layer -> UI Event Router -> **handleDataEvent()** -> PlayerRespawnPointData updated -> **Page is closed**

