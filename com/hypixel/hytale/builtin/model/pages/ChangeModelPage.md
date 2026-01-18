---
description: Architectural reference for ChangeModelPage
---

# ChangeModelPage

**Package:** com.hypixel.hytale.builtin.model.pages
**Type:** Transient

## Definition
```java
// Signature
public class ChangeModelPage extends InteractiveCustomUIPage<ChangeModelPage.PageEventData> {
```

## Architecture & Concepts
The ChangeModelPage class is a server-side controller that manages the user interface for a player to select and preview a new character model. It is a direct implementation of the server-authoritative UI pattern, where the client's user interface is a thin view driven entirely by commands from the server.

This class acts as the bridge between player input from the client, the server's asset database (ModelAsset), and the Entity Component System (Store). When active, it dynamically generates a list of available models, handles search queries, and spawns a temporary preview entity in the world for the player to inspect.

It extends InteractiveCustomUIPage, indicating it's a stateful, interactive screen that responds to specific, structured events defined by the nested PageEventData class. The core responsibility is to translate low-level UI events (button clicks, text input) into high-level game state changes, such as updating the player's actual ModelComponent or cleaning up the preview entity.

### Lifecycle & Ownership
- **Creation:** Instantiated and assigned to a player when they initiate the model change process, typically via a command or another UI interaction. The page is registered with the player's PageManager, which formally begins its lifecycle.
- **Scope:** The object's lifetime is strictly bound to the time the player has the "Change Model" UI open. All its internal state, such as the search query and the reference to the preview entity, is specific to that single UI session.
- **Destruction:** The object is marked for garbage collection when the UI is dismissed. This occurs when the player confirms a model selection (which sets the page to Page.None), manually closes the window, or disconnects. The onDismiss method is a critical destructor hook that ensures the preview entity is removed from the world, preventing entity leaks.

## Internal State & Concurrency
- **State:** This class is highly stateful and mutable. It maintains the current search query, the filtered list of models displayed to the user, the currently selected model, and a direct reference (Ref) to the temporary preview entity within the ECS. It also caches the position, rotation, and scale for the preview model.
- **Thread Safety:** This class is **not thread-safe**. All interactions with it, especially methods that modify the Entity Store (Store), are expected to occur on the main server game thread. Unsynchronized access from other threads will lead to race conditions, ECS corruption, and unpredictable client-side behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, commandBuilder, eventBuilder, store) | void | O(N) | Called once by the framework to construct the initial UI. Binds core event listeners. Complexity is proportional to the number of models (N) being listed. |
| handleDataEvent(ref, store, data) | void | Varies | The primary event router. Handles all incoming UI events from the client, such as search queries, model selections, and scale changes. Complexity depends on the event type. |
| onDismiss(ref, store) | void | O(1) | Lifecycle hook called when the page is closed. Responsible for cleaning up world-state resources, specifically the preview entity. |

## Integration Patterns

### Standard Usage
This page should be instantiated and managed exclusively through the player's PageManager. The server initiates the UI session, and the framework then drives the page's lifecycle by invoking its build and event handling methods in response to network packets.

```java
// A server system (e.g., a command handler) opens the page for a player.
Player player = store.getComponent(playerRef, Player.getComponentType());
if (player != null) {
    player.getPageManager().setPage(playerRef, store, new ChangeModelPage(playerRef));
}
```

### Anti-Patterns (Do NOT do this)
- **Leaking Preview Entities:** Overriding the onDismiss method without calling super.onDismiss() or otherwise failing to remove the modelPreview entity will permanently leak entities into the world.
- **External State Mutation:** Modifying the internal state of this class (e.g., searchQuery, selectedModel) from outside the handleDataEvent flow will cause a desynchronization between the server's state and what the client UI is displaying.
- **Cross-Player Usage:** An instance of ChangeModelPage is tied to a single player (via its PlayerRef). Attempting to reuse an instance for multiple players is not supported and will lead to severe state corruption.

## Data Pipeline
The flow of data is bidirectional, originating from either the server building the UI or the client sending an event.

> **UI Initialization Flow:**
> Server Logic -> `PageManager.setPage()` -> **ChangeModelPage.build()** -> `UICommandBuilder` / `UIEventBuilder` -> Network Packet -> Client Renders UI

> **Player Interaction Flow (e.g., Search):**
> Client Input -> UI Event Packet -> Server Network Layer -> **ChangeModelPage.handleDataEvent()** -> `buildModelList()` -> `UICommandBuilder` Update -> Network Packet -> Client UI List Updates

> **Preview Entity Flow:**
> Client Clicks Model -> UI Event Packet -> **ChangeModelPage.handleDataEvent()** -> `selectModel()` -> `store.addEntity()` / `store.putComponent()` -> ECS Update -> Entity appears/changes in player's client view

