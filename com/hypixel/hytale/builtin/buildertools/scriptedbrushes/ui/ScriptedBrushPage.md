---
description: Architectural reference for ScriptedBrushPage
---

# ScriptedBrushPage

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.ui
**Type:** Transient

## Definition
```java
// Signature
public class ScriptedBrushPage extends InteractiveCustomUIPage<FileBrowserEventData> {
```

## Architecture & Concepts
The ScriptedBrushPage class is a server-side controller for a specific client-side user interface. It acts as the bridge between a player's UI interactions and the server's builder tool systems. Architecturally, it is not a standalone component but rather a specialized implementation within the server's broader UI framework, inheriting from InteractiveCustomUIPage.

Its primary responsibility is to present a player with a searchable list of available ScriptedBrushAssets. To achieve this, it employs a composition pattern, encapsulating and configuring a generic **ServerFileBrowser** component. This design decouples the general UI logic of a file browser (searching, listing, event handling) from the specific data source it needs to display.

The data source itself is provided through a Strategy pattern, implemented by the nested private class **ScriptedBrushListProvider**. This provider is responsible for querying the global ScriptedBrushAsset map, performing fuzzy-string searches, and returning a list of entries to the ServerFileBrowser for rendering.

Upon player selection, the page modifies the player's state by loading the chosen brush configuration into their PrototypePlayerBuilderToolSettings.

### Lifecycle & Ownership
- **Creation:** An instance of ScriptedBrushPage is created on-demand by the server, typically in response to a player action such as executing a command or interacting with another UI element. It is instantiated with a direct reference to the target player (PlayerRef), binding the page's lifecycle to that specific player.
- **Scope:** The object's lifetime is ephemeral. It exists only as long as the player has the brush selection UI open. The `CustomPageLifetime.CanDismiss` flag passed to the superconstructor enforces this transient nature.
- **Destruction:** The instance is marked for garbage collection when the player's PageManager transitions to a different page (including Page.None) or when the player disconnects. The `handleBrushSelection` method explicitly closes the page upon a successful selection, triggering its destruction.

## Internal State & Concurrency
- **State:** The class holds mutable state, primarily within its internal ServerFileBrowser instance. This includes the current search query and the list of filtered results. This state is modified exclusively through the `handleDataEvent` method in response to client-side UI events.
- **Thread Safety:** This class is **not thread-safe** and must not be accessed from multiple threads. All interactions with a ScriptedBrushPage instance are expected to occur on the primary server thread responsible for the associated player's updates. The engine's Entity-Component-System (ECS) architecture, indicated by the use of Ref and Store parameters, ensures serialized access during its lifecycle methods.

## API Surface
The public API is minimal and primarily consists of lifecycle methods called by the server's UI framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, commandBuilder, eventBuilder, store) | void | O(1) | Called by the framework to generate the initial UI commands for rendering the page on the client. |
| handleDataEvent(ref, store, data) | void | O(N) | The primary entry point for client interaction. Processes events from the UI. Complexity is O(N) during search, where N is the total number of brushes, otherwise O(1). |

## Integration Patterns

### Standard Usage
This page should always be instantiated and managed via a player's PageManager. A server-side system, such as a command executor, is the typical integration point.

```java
// In a command handler or other server-side logic
Player player = ...; // Get the target player component
PlayerRef playerRef = ...; // Get the player's reference
Store<EntityStore> store = ...; // Get the entity store

// Create and assign the page to the player
ScriptedBrushPage brushPage = new ScriptedBrushPage(playerRef);
player.getPageManager().setPage(ref, store, brushPage);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation without Assignment:** Creating `new ScriptedBrushPage()` without immediately assigning it to a player's PageManager is a memory leak. The object will have no connection to the UI framework and will never be used or garbage collected correctly.
- **State Tampering:** Do not attempt to access or modify the internal `browser` field from outside the class. All interactions must flow through the `handleDataEvent` method.
- **Cross-Player Reference:** A ScriptedBrushPage is inextricably linked to the PlayerRef provided in its constructor. Attempting to reuse an instance for a different player will lead to incorrect state and severe bugs.

## Data Pipeline
The component facilitates a request-response data flow between the client's UI and the server's game state.

> **Flow (Server to Client):**
> PageManager sets page -> **ScriptedBrushPage.build()** -> UICommandBuilder -> Network Packet -> Client Renders UI

> **Flow (Client to Server):**
> Player clicks brush -> Client sends FileBrowserEventData -> Network Packet -> **ScriptedBrushPage.handleDataEvent()** -> handleBrushSelection() -> Player's Tool Settings are mutated

