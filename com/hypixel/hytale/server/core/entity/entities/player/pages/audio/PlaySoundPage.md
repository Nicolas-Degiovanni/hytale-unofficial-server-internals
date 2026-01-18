---
description: Architectural reference for PlaySoundPage
---

# PlaySoundPage

**Package:** com.hypixel.hytale.server.core.entity.entities.player.pages.audio
**Type:** Transient

## Definition
```java
// Signature
public class PlaySoundPage extends InteractiveCustomUIPage<PlaySoundPage.PlaySoundPageEventData> {
```

## Architecture & Concepts
The PlaySoundPage class is a server-side Controller component within the Model-View-Controller (MVC) paradigm used by the custom UI framework. It is responsible for managing the state and interaction logic of a specific debug/developer UI page that allows a player to search for, select, and play any registered sound event in the game.

This class acts as a bridge between the client-side user interface (the View) and the server's core audio and asset systems (the Model). It receives structured event data from the client, processes it to update its internal state, and issues targeted UI update commands back to the client.

Key architectural responsibilities include:
-   **Dynamic UI Generation:** It builds and rebuilds the list of available sounds based on player search queries.
-   **State Management:** It maintains the complete state of the UI, including search text, slider values, and the currently selected sound.
-   **Event Handling:** It serves as the single entry point for all user interactions on this page, deserializing event data and routing it to the appropriate logic.
-   **Decoupling:** It decouples the UI from the core `SoundUtil` and `SoundEvent` asset registry. The page itself does not contain sound playback logic; it delegates this responsibility after validating the current state.

The generic parameter `PlaySoundPageEventData` defines the strict data contract for all communication between the client-side UI and this server-side controller, ensuring type safety and clear intent.

## Lifecycle & Ownership
-   **Creation:** An instance of PlaySoundPage is created by a higher-level UI management system when a player is directed to this specific page. The constructor requires a `PlayerRef`, binding the page's lifecycle directly to that of a player's UI session.
-   **Scope:** The object persists only as long as the player has the "Play Sound" page open. It is a short-lived, stateful object.
-   **Destruction:** The instance is eligible for garbage collection once the player closes or navigates away from the UI page, and the server's UI manager releases its reference. There is no manual destruction or cleanup method.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. It caches the player's search query, the filtered list of sound events, the currently selected sound, and the volume and pitch values from the UI sliders. This server-side state is intended to be the single source of truth, mirroring the state of the client's view.

-   **Thread Safety:** **This class is not thread-safe.** All methods are designed to be called from a single, serialized game loop thread corresponding to the player entity. The internal state fields like `searchQuery` and `selectedSoundEvent` are not protected by locks or other concurrency primitives.

    **Warning:** Accessing or modifying an instance of PlaySoundPage from any thread other than the owning player's update thread will result in race conditions, UI state corruption, and unpredictable server behavior.

## API Surface
The primary public contract is defined by its parent class, `InteractiveCustomUIPage`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, commandBuilder, eventBuilder, store) | void | O(N log N) | Constructs the initial UI state and event bindings. Complexity is dominated by sorting the full list of sound events (N). |
| handleDataEvent(ref, store, data) | void | O(M) | Processes a data event from the client. Complexity for search is O(M) where M is the total number of sound events, due to the fuzzy search algorithm. Other events are O(1). |

## Integration Patterns

### Standard Usage
This class should be instantiated and managed by a central UI service responsible for a player's interface. The service opens the page, which triggers the initial `build` call, and subsequently routes incoming UI events to the `handleDataEvent` method.

```java
// In a hypothetical PlayerUIManager class
public void showPlaySoundScreen(PlayerRef playerRef) {
    // A new page is created for each time the UI is opened
    PlaySoundPage soundPage = new PlaySoundPage(playerRef);

    // The manager would then register this page and send it to the client
    this.openCustomPageForPlayer(playerRef, soundPage);
}
```

### Anti-Patterns (Do NOT do this)
-   **State Caching:** Do not hold a reference to a PlaySoundPage instance after the player has closed the UI. Its state becomes stale and it prevents the object from being garbage collected.
-   **External State Mutation:** Do not modify the internal state (e.g., `searchQuery`, `volumeDecibels`) from outside the `handleDataEvent` method. Doing so bypasses the logic that sends UI updates to the client, causing a state desynchronization.
-   **Reusing Instances:** Do not reuse a PlaySoundPage instance for multiple players or for the same player across different sessions. The object is bound to a specific `PlayerRef` and UI session.

## Data Pipeline
The flow of data is a closed loop, originating from user input on the client and resulting in either a UI update or a world action.

**Scenario: User searches for a sound**
> Flow:
> Client UI Input (typing in search box) -> `PlaySoundPageEventData` (with `searchQuery` set) -> Network Packet -> Server Deserializer -> **handleDataEvent** -> `buildSoundEventList` -> `UICommandBuilder` -> Network Packet (with UI clear/append commands) -> Client UI Render

**Scenario: User plays a selected sound**
> Flow:
> Client UI Input (clicking "Play" button) -> `PlaySoundPageEventData` (with `type` set to "Play") -> Network Packet -> Server Deserializer -> **handleDataEvent** -> `SoundUtil.playSoundEvent2d` -> Server Audio System -> Sound broadcast to relevant clients

