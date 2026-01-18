---
description: Architectural reference for PortalDeviceActivePage
---

# PortalDeviceActivePage

**Package:** com.hypixel.hytale.builtin.portals.ui
**Type:** Transient

## Definition
```java
// Signature
public class PortalDeviceActivePage extends InteractiveCustomUIPage<PortalDeviceActivePage.Data> {
```

## Architecture & Concepts

The PortalDeviceActivePage is a server-side controller responsible for generating the user interface that a player sees when interacting with an already active Portal Device. It does not render the UI directly; instead, it functions as a state machine that queries the game world and translates the current status of a portal into a series of commands for the client-side UI engine.

This class acts as a bridge between the game's entity-component system and the UI framework. When a player opens the interface for an active portal, an instance of this class is created. On each UI tick, its **build** method is invoked. This method executes the core logic: it reads data from the world state—specifically the PortalDevice component on the block and the PortalWorld resource in the destination dimension—and uses a UICommandBuilder to assemble a payload of instructions. These instructions tell the client which UI elements to show, hide, or update, such as the portal's name, the number of players inside, and the remaining time.

This pattern decouples the UI presentation (defined in a .ui file) from the complex game state logic, allowing developers to change the UI's appearance without altering the server's state validation rules.

### Lifecycle & Ownership
-   **Creation:** An instance is created by the server's UI management system when a player successfully interacts with a block entity that has an active PortalDevice component. The constructor requires a reference to the player, the portal's configuration, and a direct reference to the portal block entity.
-   **Scope:** The object's lifetime is tied directly to the player's UI session. It exists only while the player has this specific UI page open. The lifetime is governed by the **CustomPageLifetime.CanDismissOrCloseThroughInteraction** policy, indicating it is ephemeral.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection as soon as the player closes the UI, navigates to a different screen, or is disconnected from the server.

## Internal State & Concurrency
-   **State:** This class is fundamentally stateless regarding world data. It holds immutable references to the **PortalDeviceConfig** and the **Ref<ChunkStore>** for the portal block, which are provided at construction. All other information, such as the destination world's player count or remaining time, is fetched live from the canonical world state during every invocation of the **build** method. It performs no caching of world state.
-   **Thread Safety:** **This class is not thread-safe and must not be accessed from multiple threads.** It is designed to be created, used, and destroyed exclusively on the server's main world thread for a given player. Its methods read from shared, mutable game state (ChunkStore, EntityStore), and it relies entirely on the engine's single-threaded game loop to ensure data consistency. Off-thread access will result in severe race conditions and world corruption.

## API Surface

The primary interaction point is the overridden **build** method, which is part of the InteractiveCustomUIPage contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, commandBuilder, eventBuilder, store) | void | O(1) | Populates the UICommandBuilder with instructions based on the real-time state of the associated portal. Complexity is constant time relative to system load, involving several direct lookups in world storage. Throws exceptions if critical world data is unexpectedly missing. |

## Integration Patterns

### Standard Usage

This class is not intended to be used directly by most game logic. It is instantiated and managed by the server's core UI framework in response to player input. The framework is responsible for passing the correct player and block references.

```java
// Conceptual example of how the engine might open this page for a player.
// This code would exist within a block interaction handler.

PlayerRef player = ...;
Ref<ChunkStore> portalBlockRef = ...;
PortalDeviceConfig config = ...;

// The UI manager creates and displays the page.
player.getUIManager().open(
    new PortalDeviceActivePage(player, config, portalBlockRef)
);
```

### Anti-Patterns (Do NOT do this)
-   **Manual Instantiation:** Do not call `new PortalDeviceActivePage()` from general game logic. The class is tightly coupled to the UI lifecycle and expects to be managed by the player's UI session handler.
-   **State Caching:** Do not attempt to store or cache the result of the internal **computeState** method. The state of the portal and its destination world can change between ticks, and the UI must always reflect the most current information.
-   **Re-using Instances:** A PortalDeviceActivePage instance is single-use. Do not hold a reference to it after the player has closed the UI, as the underlying block reference (**blockRef**) may become invalid.

## Data Pipeline

The flow of data for rendering this UI begins with player input and ends with an updated view on the client. This class is a critical processing step in the middle of that flow.

> Flow:
> Player interacts with Portal Block -> Server Interaction System -> **PortalDeviceActivePage Instantiation** -> `build()` method queries `ChunkStore` and `EntityStore` for portal status -> **PortalDeviceActivePage** generates UI commands -> `UICommandBuilder` serializes commands -> Network Packet sent to Client -> Client UI Engine renders the updates

