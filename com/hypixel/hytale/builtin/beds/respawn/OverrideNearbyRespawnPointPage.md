---
description: Architectural reference for OverrideNearbyRespawnPointPage
---

# OverrideNearbyRespawnPointPage

**Package:** com.hypixel.hytale.builtin.beds.respawn
**Type:** Transient

## Definition
```java
// Signature
public class OverrideNearbyRespawnPointPage extends RespawnPointPage {
```

## Architecture & Concepts
The OverrideNearbyRespawnPointPage is a server-side UI component that represents a specific, stateful user interface screen. It is part of the server-authoritative UI framework, where the server dictates the structure, content, and behavior of UI elements presented to the client.

This class is invoked in a specific gameplay scenario: when a player attempts to set a new respawn point (e.g., by interacting with a bed) in close proximity to one or more existing respawn points. Instead of silently overwriting the old point, the system presents the player with a confirmation dialog. This class is responsible for building that dialog.

Its primary functions are:
1.  **Data Aggregation:** It captures a snapshot of the game state at the moment of interaction, including the player's entity reference, the location of the new respawn point, and a list of all nearby respawn points that would be affected.
2.  **UI Construction:** The build method dynamically generates a series of UI commands. These commands instruct the client to render a specific UI template (OverrideNearbyRespawnPointPage.ui), populate it with data (like the list of nearby points and their distances), and set up event bindings for interactive elements like buttons.
3.  **Event Handling:** It defines the server-side logic to execute when the player interacts with the UI. The handleDataEvent method processes incoming events from the client, such as confirming the new respawn point with a custom name or canceling the operation.

This class acts as a short-lived controller in a Model-View-Controller pattern, where the Model is the server's entity and world state, the View is the client-side UI, and this Page is the Controller that mediates between them for a single, atomic interaction.

### Lifecycle & Ownership
-   **Creation:** An instance is created on the server by a higher-level game logic system, typically a block interaction handler. When a player's interaction with a potential respawn block is processed, the server logic calculates if other respawn points are within a defined radius. If so, it instantiates this class with the necessary context.
-   **Scope:** The object's lifetime is strictly bound to the player's current UI state. It exists only as long as the "Override Nearby Respawn Point" dialog is the active page for that player. It is a single-use object designed for one interaction.
-   **Destruction:** The object is eligible for garbage collection as soon as the player's PageManager transitions to a new page (e.g., Page.None after canceling, or another UI). There is no explicit destruction method; its lifecycle is managed entirely by the player's UI state machine.

## Internal State & Concurrency
-   **State:** This class is stateful. It holds immutable references to the game world state captured at the time of its creation (positions, nearby points, etc.). This state is used by the build method to construct the UI. It is effectively a snapshot and does not update if the underlying world state changes while the UI is open.
-   **Thread Safety:** This class is **not thread-safe** and must only be accessed from the server's main game thread. All of its methods operate on live game state via the Ref and Store parameters, which are themselves not thread-safe. All creation, method invocation, and handling must be synchronized with the server's primary update loop to prevent data corruption and race conditions.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, commandBuilder, eventBuilder, store) | void | O(N) | Populates UI builders with commands to construct the client-side dialog. N is the number of nearby respawn points. |
| handleDataEvent(ref, store, data) | void | O(1) | Executes server-side logic based on a UI event from the client, such as confirming or canceling the action. |

## Integration Patterns

### Standard Usage
This class is not meant to be used directly. It is part of the Page framework managed by the Player's PageManager. The correct integration is to instantiate it with the required context and set it as the player's active page.

```java
// In a block interaction handler...
PlayerRespawnPointData[] nearbyPoints = findNearbyPoints(player, newPosition);
int radiusLimit = getConfigValue("respawnRadiusLimit");

if (nearbyPoints.length > 0) {
    // Create the page with the necessary context
    OverrideNearbyRespawnPointPage page = new OverrideNearbyRespawnPointPage(
        playerRef,
        interactionType,
        newPosition,
        respawnBlock,
        nearbyPoints,
        radiusLimit
    );

    // The PageManager will now manage its lifecycle
    player.getPageManager().setPage(playerRef, store, page);
} else {
    // Set the respawn point directly
    setRespawnPointForPlayer(...);
}
```

### Anti-Patterns (Do NOT do this)
-   **Reusing Instances:** Do not cache and reuse an instance of this class. It contains a snapshot of the world at a specific moment. Reusing it would present the player with stale data. A new instance must be created for every interaction.
-   **Manual Method Invocation:** Do not call the build or handleDataEvent methods directly. They are callbacks designed to be invoked by the PageManager as part of the server's UI update and network event processing pipeline. Bypassing the manager will break the UI lifecycle.
-   **Long-Term Storage:** Storing a reference to this object beyond the scope of the immediate interaction is a memory leak. Its lifecycle is intentionally brief.

## Data Pipeline
The flow of data and control for this UI interaction is server-authoritative and follows a strict request-response pattern.

> Flow:
> 1.  **Player Interaction:** Client sends a packet indicating interaction with a block (e.g., a bed).
> 2.  **Server Logic:** The server's block handler receives the packet, checks for nearby respawn points, and instantiates **OverrideNearbyRespawnPointPage**.
> 3.  **Page Activation:** The instance is passed to the player's PageManager.
> 4.  **UI Generation:** The PageManager invokes the **build()** method, which generates a list of UI commands.
> 5.  **Network Transmission:** These UI commands are serialized into packets and sent to the client.
> 6.  **Client Rendering:** The client receives the packets, interprets the commands, and renders the "Override" dialog.
> 7.  **Client Event:** The player clicks a button (e.g., "Confirm"). The client sends a CustomUIEvent packet back to the server containing event data.
> 8.  **Server Handling:** The server's network layer routes the event to the player's PageManager, which calls **handleDataEvent()** on the active page instance.
> 9.  **State Mutation:** The **handleDataEvent()** method processes the event, either setting the new respawn point or closing the UI, thus modifying the player's state.

