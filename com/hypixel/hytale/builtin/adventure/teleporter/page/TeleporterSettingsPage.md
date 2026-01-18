---
description: Architectural reference for TeleporterSettingsPage
---

# TeleporterSettingsPage

**Package:** com.hypixel.hytale.builtin.adventure.teleporter.page
**Type:** Transient

## Definition
```java
// Signature
public class TeleporterSettingsPage extends InteractiveCustomUIPage<TeleporterSettingsPage.PageEventData> {
```

## Architecture & Concepts
The TeleporterSettingsPage class is a server-side controller that manages the user interface for configuring a specific in-world teleporter block. It follows a remote MVC (Model-View-Controller) pattern where the server holds the model and controller logic, while the client is responsible for rendering the view based on commands sent by this class.

This class does not render UI directly. Instead, its primary function is to translate the state of a Teleporter component into a series of declarative commands for the client. This is achieved through the `UICommandBuilder` during the `build` phase. These commands instruct the client on how to populate and display the `Pages/Teleporter.ui` file, setting values for input fields, populating dropdowns, and managing the visibility of different UI panels.

Player interactions on the client, such as clicking the save button, are bound to server-side logic via the `UIEventBuilder`. When a bound event is triggered, the client serializes the state of specified UI elements into a `PageEventData` payload and sends it to the server. The `handleDataEvent` method then acts as the event handler, deserializing this payload, validating the input, and applying the changes to the world state.

The class operates in one of two modes:
*   **FULL:** Provides a comprehensive interface for setting a teleporter's destination coordinates, rotation, target world, and linked warp point.
*   **WARP:** Provides a simplified interface for linking to an existing warp point or creating a new one.

This clear separation between UI construction (`build`) and event handling (`handleDataEvent`) makes it a robust and central component for managing teleporter configuration.

## Lifecycle & Ownership
-   **Creation:** An instance of TeleporterSettingsPage is created on-demand when a player interacts with a teleporter block in the world. The system handling the interaction is responsible for instantiating this class, providing it with the context of the specific player (`PlayerRef`) and the target teleporter block entity (`Ref<ChunkStore>`).
-   **Scope:** The object's lifetime is strictly bound to the player's UI session. It exists only as long as the settings page is the active page for that player.
-   **Destruction:** The instance is discarded and becomes eligible for garbage collection as soon as the player closes the UI or the server programmatically changes the player's active page via `PageManager.setPage`. There are no persistent references to this object after the interaction concludes.

## Internal State & Concurrency
-   **State:** The internal state of a TeleporterSettingsPage instance, such as the reference to the teleporter block and the UI mode, is defined at construction and is effectively immutable. The class is designed to be stateless regarding the UI itself; it reads directly from the world state (Teleporter component, Universe worlds, TeleportPlugin warps) during the `build` process and writes back to the world state in `handleDataEvent`. It performs no internal caching of this data.
-   **Thread Safety:** **This class is not thread-safe and must not be accessed from multiple threads.** All interactions with a Page instance are expected to occur on the dedicated entity processing thread for the associated player. The engine guarantees this contract for all standard UI interactions. Asynchronous modification or access would lead to severe race conditions when reading from or writing to the world state.

## API Surface
The public contract is defined by its constructor and the methods overridden from its parent class, `InteractiveCustomUIPage`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| TeleporterSettingsPage(player, block, mode, state) | constructor | O(1) | Creates a new UI page controller for a specific player and teleporter block. |
| build(ref, commandBuilder, eventBuilder, store) | void | O(W+V) | Populates the command and event builders to construct the client-side UI. Complexity is proportional to the number of worlds (W) and available warps (V) in the game. |
| handleDataEvent(ref, store, data) | void | O(1) | Processes incoming UI event data from the client. Performs validation, updates the Teleporter component, and modifies the global warp registry. Throws assertions if component state is invalid. |

## Integration Patterns

### Standard Usage
This class is not intended to be used in isolation. It is created and managed by a higher-level system that processes player interactions with blocks. The typical flow involves setting it as the player's active page.

```java
// In a system that handles player interaction with a teleporter block:
Player player = store.getComponent(playerEntityRef, Player.getComponentType());
Ref<ChunkStore> teleporterBlockRef = ... // Get reference to the teleporter block entity

// Create and display the full settings page
TeleporterSettingsPage page = new TeleporterSettingsPage(
    player.getRef(),
    teleporterBlockRef,
    TeleporterSettingsPage.Mode.FULL,
    "active_state_name"
);

player.getPageManager().setPage(playerEntityRef, store, page);
```

### Anti-Patterns (Do NOT do this)
-   **Re-using Instances:** Do not cache and re-use a TeleporterSettingsPage instance. A new instance must be created for every unique player interaction to ensure it has the correct, up-to-date context.
-   **Manual Event Handling:** Do not attempt to call `handleDataEvent` directly. This method is exclusively for the server's UI event processing pipeline, which handles deserialization and routing.
-   **Access Outside Player Tick:** Do not hold a reference to this page and access it from other systems or threads. Its lifecycle is ephemeral and tied strictly to the player's main update loop.

## Data Pipeline
The flow of data for this UI is a complete round-trip, from the server generating the view to the server processing the user's input.

> **UI Construction Flow:**
> Player Interaction -> Server System -> **new TeleporterSettingsPage()** -> `build()` method reads from `Teleporter` component & `TeleportPlugin` -> `UICommandBuilder` -> Network Packet -> Client Renders UI

> **UI Event Flow:**
> Client User Input (e.g., Save Click) -> `PageEventData` Payload -> Network Packet -> Server UI System -> **TeleporterSettingsPage.handleDataEvent()** -> `Teleporter` component & `TeleportPlugin` updated -> Player Page Closed

