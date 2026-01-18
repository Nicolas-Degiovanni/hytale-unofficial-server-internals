---
description: Architectural reference for HudManager
---

# HudManager

**Package:** com.hypixel.hytale.server.core.entity.entities.player.hud
**Type:** Transient State Object

## Definition
```java
// Signature
public class HudManager {
```

## Architecture & Concepts
The HudManager is a server-side component responsible for managing the state of a single player's Heads-Up Display. It does not perform any rendering; instead, it acts as an authoritative state machine that synchronizes its state with the player's client via network packets. Each player entity possesses its own unique instance of HudManager.

Its core responsibilities are twofold:
1.  **Standard Component Management:** Tracking the visibility of predefined, modular UI elements such as the hotbar, health indicator, and compass. These are represented by the HudComponent enum.
2.  **Custom UI Management:** Handling the lifecycle of entirely custom, scriptable user interfaces, represented by the CustomUIHud class.

All state modifications within this class are designed to be transactional from the client's perspective. Any method call that alters the HUD's state immediately dispatches a corresponding network packet to the client, ensuring the UI is updated in near real-time. This class forms a critical bridge between server-side game logic and client-side presentation.

## Lifecycle & Ownership
-   **Creation:** A new HudManager instance is created for each player entity upon their initialization in the game world. It is a fundamental component of the player's server-side representation. The default constructor populates the manager with a standard set of visible UI components.
-   **Scope:** The object's lifetime is strictly bound to the player's session. It persists as long as the player entity exists on the server.
-   **Destruction:** The HudManager is eligible for garbage collection when its parent player entity is destroyed, typically when the player disconnects from the server. There is no explicit cleanup or `destroy` method.

## Internal State & Concurrency
-   **State:** The internal state is highly mutable. It maintains two primary pieces of state:
    1.  A `Set<HudComponent>` named visibleHudComponents, which tracks all standard UI elements currently visible to the player.
    2.  A nullable reference to a `CustomUIHud`, representing a completely custom UI overlay.

-   **Thread Safety:** This class is **partially thread-safe**.
    -   The set of visible components, `visibleHudComponents`, is backed by a `ConcurrentHashMap.newKeySet`, making additions and removals of standard components from multiple threads safe. This is a defensive design to prevent corruption if, for example, a quest system and a combat system attempt to modify the HUD simultaneously.
    -   **WARNING:** The `customHud` field is not volatile and its access is not synchronized. Concurrent calls to `setCustomHud` from different threads will result in a race condition. All interactions involving custom UIs should be marshaled onto a single, player-specific thread or the main server thread to ensure state consistency.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setVisibleHudComponents(ref, components) | void | O(N) | Atomically replaces the entire set of visible components with the provided set. Dispatches an `UpdateVisibleHudComponents` packet. |
| showHudComponents(ref, components) | void | O(N) | Adds one or more components to the visible set. Dispatches an `UpdateVisibleHudComponents` packet. |
| hideHudComponents(ref, components) | void | O(N) | Removes one or more components from the visible set. Dispatches an `UpdateVisibleHudComponents` packet. |
| setCustomHud(ref, hud) | void | O(1) | Sets or clears the active custom UI. Dispatches a `CustomHud` packet to the client. **Not thread-safe.** |
| resetHud(ref) | void | O(N) | Resets the player's HUD to the default state, showing all standard components and clearing any custom UI. |
| resetUserInterface(ref) | void | O(1) | Sends a special packet to the client, forcing a hard reset of its entire UI state. This is a more drastic action than `resetHud`. |

## Integration Patterns

### Standard Usage
The HudManager should always be retrieved from its parent Player entity. Logic should then be invoked to manipulate the HUD state, passing in the player's reference to enable packet dispatching.

```java
// Assumes 'player' is the server-side Player object
PlayerRef playerRef = player.getRef();
HudManager hudManager = player.getHudManager();

// Hide the compass and reticle during a cutscene
hudManager.hideHudComponents(playerRef, HudComponent.Compass, HudComponent.Reticle);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new HudManager()`. The manager is part of the Player's state and must be managed by the entity's lifecycle. Direct instantiation creates an orphan object that is not linked to any player's network connection.
-   **State Caching:** Do not cache the result of `getVisibleHudComponents()` for later use. The internal state can be changed by other systems at any time. Always retrieve the manager from the player and call the appropriate method.
-   **Concurrent Custom UI Modification:** Do not call `setCustomHud` from multiple threads without external synchronization. This can lead to race conditions where the server's state and the client's state become desynchronized.

## Data Pipeline
The HudManager acts as a stateful controller in the server-to-client data flow for UI updates.

> Flow:
> Server-Side Game Logic (e.g., Quest System, Combat Handler) -> Player.getHudManager().show(...) -> **HudManager** (Internal state is mutated) -> PacketHandler.writeNoCache(new UpdateVisibleHudComponents(...)) -> Network Layer -> Client UI System -> Render Update

