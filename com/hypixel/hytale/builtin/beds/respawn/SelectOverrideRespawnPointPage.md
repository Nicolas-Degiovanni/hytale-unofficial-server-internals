---
description: Architectural reference for SelectOverrideRespawnPointPage
---

# SelectOverrideRespawnPointPage

**Package:** com.hypixel.hytale.builtin.beds.respawn
**Type:** Transient

## Definition
```java
// Signature
public class SelectOverrideRespawnPointPage extends RespawnPointPage {
```

## Architecture & Concepts
The SelectOverrideRespawnPointPage class is a server-side component that functions as a stateful controller for a specific user interface screen. Within Hytale's UI framework, a Page represents the logic and data for a single, interactive view presented to the client. This particular implementation manages the screen shown when a player attempts to set a new respawn point (e.g., by using a bed) but has already reached their maximum number of saved points.

Its core responsibility is to:
1.  Receive the list of the player's existing respawn points.
2.  Dynamically generate UI commands to render this list for the client, including metadata like distance and relative direction.
3.  Process incoming UI events from the client, such as button clicks to select a respawn point to override.
4.  Maintain the transient state of the UI, specifically which respawn point is currently selected by the player.
5.  Orchestrate the final action of either confirming the override or canceling the operation.

This class acts as a bridge between the core game logic (player data and world interaction) and the client's user interface, ensuring the UI is a direct reflection of server-side state and that player input is handled authoritatively by the server.

### Lifecycle & Ownership
-   **Creation:** An instance is created on-demand by server-side game logic, specifically when a player interacts with a RespawnBlock and the system determines an override is necessary. The constructor is populated with the player's current respawn data and the context of the new respawn location.
-   **Scope:** The object's lifetime is extremely short and is tied directly to the player's interaction session with this specific UI screen. It is created, used for a few seconds or minutes, and then discarded.
-   **Destruction:** The instance becomes eligible for garbage collection as soon as the player's active page is changed by the PageManager. This occurs when the player confirms their choice, cancels, or disconnects. No long-term references are held to this object.

## Internal State & Concurrency
-   **State:** This class is highly mutable. It maintains the list of respawn points passed during construction and, most importantly, the `selectedRespawnPointIndex`, which is updated in response to player input. This state is not persisted and is lost when the object is destroyed.
-   **Thread Safety:** This class is **not thread-safe** and must be considered thread-hostile. It is designed to be accessed and modified exclusively by the server's main game loop thread for a single player. All methods implicitly assume they are operating in a sequential, single-threaded context for that player's state.

**WARNING:** Accessing an instance of this class from any thread other than the primary server thread for the associated player will result in race conditions, inconsistent UI state, and potential server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, commandBuilder, eventBuilder, store) | void | O(N) | Called by the PageManager to generate the initial UI commands. Iterates through all N respawn points to build the list. |
| handleDataEvent(ref, store, data) | void | O(1) | The primary entry point for client-side UI events. Dispatches logic based on the event data, such as selecting a point or confirming the action. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most developers. It is instantiated and managed by the core respawn system. The system sets this page on a player's PageManager, which then drives the lifecycle by invoking the `build` and `handleDataEvent` methods.

```java
// Conceptual example of how the system uses this class
Player player = ...;
PlayerRespawnPointData[] existingPoints = player.getRespawnData();
Vector3i newPointPosition = ...;
RespawnBlock newPointBlock = ...;

// The system creates the page and sets it as active
SelectOverrideRespawnPointPage page = new SelectOverrideRespawnPointPage(
    player.getRef(),
    interactionType,
    newPointPosition,
    newPointBlock,
    existingPoints
);

player.getPageManager().setPage(player.getRef(), store, page);
```

### Anti-Patterns (Do NOT do this)
-   **State Caching:** Do not retain a reference to a SelectOverrideRespawnPointPage instance after it has been replaced in the PageManager. Its internal state is transient and will not reflect subsequent game state changes.
-   **Manual UI Updates:** Do not attempt to call methods like `setSelectedRespawnPoint` from outside the `handleDataEvent` flow. The class manages its own state updates in response to authoritative client events.

## Data Pipeline
The flow of data for this UI interaction is a clear request-response loop between the client and server, mediated by this class.

> **Flow 1: Initial Page Load**
> Player Interaction (use bed) -> Server Logic determines override needed -> **new SelectOverrideRespawnPointPage()** -> PageManager calls `build()` -> UICommandBuilder populates with respawn list -> Network Packet -> Client Renders UI

> **Flow 2: Player Selects an Item**
> Client Clicks Respawn Button -> Network Packet (CustomUIEvent) -> Server dispatches to `handleDataEvent()` -> **SelectOverrideRespawnPointPage** updates `selectedRespawnPointIndex` -> `sendUpdate()` is called -> Network Packet with style change -> Client highlights selected button

> **Flow 3: Player Confirms**
> Client Clicks Confirm Button -> Network Packet (CustomUIEvent) -> Server dispatches to `handleDataEvent()` -> **SelectOverrideRespawnPointPage** calls `setRespawnPointForPlayer()` -> Player Data is modified -> PageManager sets page to `Page.None` -> Network Packet -> Client closes UI

