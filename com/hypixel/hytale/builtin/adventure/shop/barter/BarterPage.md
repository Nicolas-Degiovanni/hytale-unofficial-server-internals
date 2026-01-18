---
description: Architectural reference for BarterPage
---

# BarterPage

**Package:** com.hypixel.hytale.builtin.adventure.shop.barter
**Type:** Transient

## Definition
```java
// Signature
public class BarterPage extends InteractiveCustomUIPage<BarterPage.BarterEventData> {
```

## Architecture & Concepts
The BarterPage class is a server-side controller that manages the state and logic for a player's interaction with a bartering shop user interface. It acts as a bridge between the client-side UI, defined in external UI template files, and the server's core game systems.

As a subclass of InteractiveCustomUIPage, it integrates directly into the server's player UI management framework. Its primary responsibility is to dynamically generate UI commands based on game state and to process incoming UI events from the player.

This class orchestrates several distinct systems:
*   **BarterShopAsset**: The static data definition for a shop, including its list of available trades. BarterPage reads from this asset to determine what to display.
*   **BarterShopState**: A global resource that manages the dynamic state of all shops, such as current stock levels and restock timers. BarterPage queries and modifies this state to reflect trades.
*   **Player Inventory**: It directly inspects the player's inventory to verify if they can afford a trade and executes item transactions upon a successful barter.
*   **UI Builder API**: It uses UICommandBuilder and UIEventBuilder to construct network packets that instruct the client on how to render and update the UI.

The design is fundamentally reactive. The `build` method is invoked by the framework to construct the initial UI, and the `handleDataEvent` method is the callback that responds to player actions like clicking a "Trade" button.

### Lifecycle & Ownership
-   **Creation:** An instance of BarterPage is created on the server whenever a player initiates an interaction that opens a specific barter shop. This is typically triggered by an interaction script or game logic which then pushes the new page onto the player's UI stack.
-   **Scope:** The object's lifetime is strictly tied to the visibility of the UI on the client. It persists only as long as the player has the specific shop interface open.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection as soon as the player closes the UI page, either by pressing a close button, hitting the escape key, or having another UI screen take precedence. The `CustomPageLifetime.CanDismissOrCloseThroughInteraction` configuration enforces this transient behavior.

## Internal State & Concurrency
-   **State:** BarterPage is stateful, holding an immutable reference to the `shopAsset` that defines its content. However, it is designed to be stateless regarding dynamic game data. It does not cache player inventory counts or shop stock levels. Instead, it fetches this data directly from the authoritative sources (Player component, BarterShopState) during each `build` or `handleDataEvent` call to ensure data consistency and prevent desynchronization.
-   **Thread Safety:** This class is **not thread-safe** and must only be accessed from the main server thread. All interactions with the Entity Component System (via the `Store` parameter) and other game state managers are predicated on a single-threaded execution model. Off-thread access will lead to race conditions and world state corruption.

## API Surface
The public API is primarily intended for consumption by the server's UI framework, not for direct developer invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, commandBuilder, eventBuilder, store) | void | O(N) | Framework callback to construct the UI. Iterates through N trades, querying game state for each. |
| handleDataEvent(ref, store, data) | void | O(M) | Framework callback to process a player action. Validates and executes a single trade with M input items. |

## Integration Patterns

### Standard Usage
The BarterPage is not meant to be managed directly. The engine's UI system instantiates it and invokes its lifecycle methods. A developer's only interaction is typically to create and register the page for a player.

```java
// In some server-side interaction handler...
PlayerRef playerRef = ...;
String shopIdToOpen = "village_blacksmith";

// The framework handles the rest of the lifecycle.
player.getUIPageManager().open(new BarterPage(playerRef, shopIdToOpen));
```

### Anti-Patterns (Do NOT do this)
-   **Manual Lifecycle Invocation:** Do not call `build` or `handleDataEvent` directly. These methods are callbacks managed by the UI framework and depend on specific engine state to function correctly.
-   **State Caching:** Do not extend this class to cache player inventory or shop stock. The stateless-by-design approach of re-fetching data from the source of truth is critical for preventing data inconsistency bugs.
-   **Cross-Thread Access:** Never share a BarterPage instance across threads or attempt to modify it from an asynchronous task. All operations must be synchronized with the main server tick.

## Data Pipeline
The flow of data is bidirectional, moving from server to client for rendering, and from client to server for interactions.

> **UI Rendering Flow:**
> Server Game Logic -> `new BarterPage()` -> UI Framework calls `build()` -> **BarterPage** reads `BarterShopState` & `Player Inventory` -> `UICommandBuilder` generates UI description -> Network Packet -> Client Renders UI

> **Player Interaction Flow:**
> Client Button Click -> Network Packet with `BarterEventData` -> UI Framework calls `handleDataEvent()` -> **BarterPage** validates trade -> Modifies `Player Inventory` & `BarterShopState` -> `UICommandBuilder` generates partial UI update -> Network Packet -> Client Updates UI

