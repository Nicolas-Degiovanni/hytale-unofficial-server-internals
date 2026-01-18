---
description: Architectural reference for RespawnPointPage
---

# RespawnPointPage

**Package:** com.hypixel.hytale.builtin.beds.respawn
**Type:** Transient

## Definition
```java
// Signature
public abstract class RespawnPointPage extends InteractiveCustomUIPage<RespawnPointPage.RespawnPointEventData> {
```

## Architecture & Concepts

The RespawnPointPage is an abstract, server-side controller that manages the user interface for creating, updating, and managing player respawn points. It serves as the foundational logic for any UI that needs to interact with the respawn system, such as interacting with a bed.

This class acts as a critical bridge between a player's client-side UI interaction and the server's persistent state. It does not render any UI itself; rather, it defines the contract and provides the core business logic for subclasses to build a UI and process player input. When a player interacts with an object like a bed, a concrete implementation of this class is instantiated to orchestrate the experience.

Its primary responsibilities include:
1.  Providing a framework for building a custom UI via the UICommandBuilder.
2.  Handling incoming UI events from the client, decoded into a RespawnPointEventData object.
3.  Executing the core logic to validate and persist a new respawn point. This involves modifying both the player's entity data (PlayerWorldData) and the world's block entity data (RespawnBlock).
4.  Managing the lifecycle of the UI page itself, closing it upon successful completion or cancellation.

## Lifecycle & Ownership

-   **Creation:** A concrete subclass of RespawnPointPage is instantiated by a server-side interaction handler when a player triggers a specific game action (e.g., right-clicking a bed). It is immediately associated with the player's session via their PageManager.
-   **Scope:** The object's lifetime is strictly bound to the duration of the UI interaction. It exists only as long as it is the active page in the player's PageManager. It is a short-lived, single-purpose object.
-   **Destruction:** The instance is marked for garbage collection as soon as the player's active page is changed. This occurs programmatically on success (via a call to `player.getPageManager().setPage(..., Page.None)`) or when the player manually dismisses the UI.

## Internal State & Concurrency

-   **State:** This class is stateful, holding a reference to the player (PlayerRef) for the duration of the interaction. However, it does not cache world or entity state. All operations fetch data directly from the canonical EntityStore and ChunkStore to ensure data consistency. The state it modifies is entirely external to the class itself, primarily within PlayerWorldData and RespawnBlock components.
-   **Thread Safety:** **CRITICAL WARNING:** This class is not thread-safe and must only be accessed from the main server thread for the corresponding world. All interactions with the world, entity stores, and player components assume a single-threaded execution model. Any attempt to invoke its methods from an asynchronous task or worker thread will result in race conditions, data corruption, and server instability.

## API Surface

As an abstract class, its primary API is the protected contract for subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setRespawnPointForPlayer(...) | void | O(N) | The core business logic handler. Validates input and persists the respawn point to the player and world state. Complexity is O(N) where N is the number of the player's existing respawn points. |
| displayError(Message) | void | O(1) | Sends a non-fatal error message to be displayed in the player's current UI without closing the page. Involves a network call. |

## Integration Patterns

### Standard Usage

A concrete subclass must be created to implement the abstract `build` method. This implementation defines the UI layout and event bindings. The page's event handler will receive a RespawnPointEventData object from the client and use its contents to invoke the base class logic.

```java
// In a concrete subclass like SetRespawnPointPage...

@Override
public void onEvent(Ref<EntityStore> ref, Store<EntityStore> store, RespawnPointEventData eventData) {
    // Assumes 'blockPosition' and 'respawnBlock' are fields on the subclass
    if ("SetName".equals(eventData.getAction())) {
        String name = eventData.getRespawnPointName();
        
        // Call the base class method to perform the actual state modification
        setRespawnPointForPlayer(ref, store, this.blockPosition, this.respawnBlock, name);
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not manually create an instance with `new`. The page must be instantiated and managed exclusively through the player's `PageManager` to function correctly.
-   **State Caching:** Do not cache references to player components or world chunks as member variables. The underlying data in the `Store` is the single source of truth and must be fetched within the method scope where it is used.
-   **Ignoring Threading:** Do not call `setRespawnPointForPlayer` from a separate thread. All logic must execute on the world's main tick thread.

## Data Pipeline

The flow of data and control for setting a respawn point is a complete round-trip from player action to state change.

> Flow:
> Player Interaction (Use Bed) -> Server Interaction System -> Instantiates `ConcreteRespawnPointPage` -> `Player.PageManager` sends UI layout to Client -> Player inputs name and clicks "Save" -> Client sends `RespawnPointEventData` packet to Server -> **RespawnPointPage**.onEvent -> **RespawnPointPage**.setRespawnPointForPlayer -> Modifies `PlayerWorldData` & `RespawnBlock` components -> Closes UI Page -> Sends confirmation `Message` to Player

