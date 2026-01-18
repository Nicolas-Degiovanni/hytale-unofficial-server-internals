---
description: Architectural reference for RespawnPage
---

# RespawnPage

**Package:** com.hypixel.hytale.server.core.entity.entities.player.pages
**Type:** Transient

## Definition
```java
// Signature
public class RespawnPage extends InteractiveCustomUIPage<RespawnPage.RespawnPageEventData> {
```

## Architecture & Concepts
The RespawnPage class is a server-side controller that manages the user interface state for a player who has died. It acts as a bridge between the server's core death mechanics (managed by the DeathComponent) and the client's user interface.

This class is not a simple data container; it is an active, stateful component responsible for a distinct phase of the player lifecycle. Its primary responsibilities are:
1.  **UI Construction:** Dynamically generating a series of UI commands based on the context of the player's death (e.g., reason for death, items lost).
2.  **Event Handling:** Defining and processing the specific user interactions allowed on the death screen, primarily the "Respawn" action.
3.  **State Transition:** Triggering the final game state change (player respawn) upon successful UI interaction and dismissal.

By encapsulating this logic, RespawnPage decouples the high-level player state machine from the low-level details of UI rendering and network communication. It is a concrete implementation of the server-driven UI pattern, where the server dictates both the layout and the interactive behavior of the client's interface.

## Lifecycle & Ownership
-   **Creation:** A RespawnPage is instantiated by a higher-level game system when a player's death is processed. This typically occurs within a system that observes entities gaining a DeathComponent. All necessary context, such as the death reason and item loss details, must be provided at construction.
-   **Scope:** The object's lifetime is strictly bound to the duration the player is viewing the death screen. It is held as the "active page" by the player's PageManager.
-   **Destruction:** The instance is marked for garbage collection when the player's PageManager transitions to a different page, which is triggered by the RespawnPage itself in the handleDataEvent method. The onDismiss method serves as its de-facto destructor, executing critical cleanup and state transition logic before the object is fully discarded.

## Internal State & Concurrency
-   **State:** The object is effectively immutable after construction. Its internal fields, such as deathReason and itemsLostOnDeath, are initialized once and are treated as read-only parameters for the UI build process. It does not cache external data or modify its own state after initialization.
-   **Thread Safety:** This class is **not thread-safe** and must not be accessed from multiple threads. It is designed to be created, managed, and invoked exclusively by the server's main game thread within the context of the Entity Component System (ECS). All interactions with the world state are safely mediated through the Store parameter passed into its methods, which guarantees transactional integrity.

## API Surface
The public contract is defined by its inheritance from InteractiveCustomUIPage.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, commandBuilder, eventBuilder, store) | void | O(N) | Populates the UI builders with commands to construct the client-side death screen. Complexity is O(N) where N is the number of unique item stacks lost. |
| handleDataEvent(ref, store, data) | void | O(1) | Callback for UI events from the client. Triggers the page dismissal process when the "Respawn" action is received. |
| onDismiss(ref, store) | void | O(1) | Lifecycle hook. Executes the final respawn logic by removing the DeathComponent from the player entity. Contains critical validation. |

## Integration Patterns

### Standard Usage
RespawnPage is not used in isolation. It must be instantiated and immediately passed to a player's PageManager to take control of the client's UI.

```java
// Within a system that handles player death
DeathItemLoss itemLoss = calculateItemLoss(player);
Message reason = getDeathReason(player);

// Create and set the page, transferring UI control
playerComponent.getPageManager().setPage(
    playerRef,
    store,
    new RespawnPage(playerRef, reason, true, itemLoss)
);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Invocation:** Do not call methods like onDismiss or handleDataEvent directly. These are lifecycle callbacks managed by the PageManager framework in response to network events and state changes. Manual invocation will break the player state machine.
-   **State Reuse:** Do not hold a reference to a RespawnPage instance after it has been dismissed. It is a single-use object tied to a specific death event.
-   **Asynchronous Respawn:** Do not attempt to respawn a player by directly removing the DeathComponent from another thread. The entire death and respawn flow must be orchestrated through this UI page to ensure client and server states remain synchronized.

## Data Pipeline
The class facilitates a round-trip data flow between the server's game logic and the client's UI.

> **Outbound Flow (Server to Client):**
> Player Death Event -> Death System creates **RespawnPage** -> PageManager invokes `build()` -> UICommandBuilder -> Network Packet (UI Commands) -> Client Renders Death Screen

> **Inbound Flow (Client to Server):**
> User clicks "Respawn" button -> Client UI Event -> Network Packet (UI Event) -> Server decodes to `RespawnPageEventData` -> PageManager routes to **RespawnPage**.`handleDataEvent()` -> **RespawnPage**.`onDismiss()` -> ECS Store removes `DeathComponent`

