---
description: Architectural reference for PortalDeviceSummonPage
---

# PortalDeviceSummonPage

**Package:** com.hypixel.hytale.builtin.portals.ui
**Type:** Transient

## Definition
```java
// Signature
public class PortalDeviceSummonPage extends InteractiveCustomUIPage<PortalDeviceSummonPage.Data> {
```

## Architecture & Concepts
The PortalDeviceSummonPage class is a server-side controller that manages the user interface for activating a Portal Device. It is not a persistent entity but rather a temporary, stateful object created in response to a specific player interaction. Its primary role is to serve as a bridge between a player's intent to open a portal and the complex, asynchronous backend processes required to create a new world instance.

This class follows a state-validation-driven UI pattern. Upon creation, it performs a series of rigorous checks in its `computeState` method to determine if a portal can be spawned. This includes verifying the item held by the player, checking global portal limits, and ensuring the target block is a valid, inactive Portal Device.

Based on the validation outcome, the `build` method dynamically constructs one of two user interfaces:
1.  **Success UI:** A detailed page (`Pages/PortalDeviceSummon.ui`) populated with data from the `PortalType` asset, including its name, description, objectives, and time limits. This page presents the player with a "Summon" button.
2.  **Error UI:** A generic error page (`Pages/PortalDeviceError.ui`) with a localized message explaining the reason for failure, such as holding the wrong item or the server reaching its portal capacity.

When the player interacts with the Success UI, the `handleDataEvent` method processes the client-side event. A click on the "Summon" button triggers the core logic: a multi-stage, asynchronous pipeline that spawns a new world instance, configures it, creates a return portal within it, and finally updates the state of the original Portal Device block.

## Lifecycle & Ownership
- **Creation:** An instance of PortalDeviceSummonPage is created by the server's gameplay logic when a player interacts with a Portal Device block while holding a potential key item. It is immediately assigned to the player's `PageManager`.

- **Scope:** The object's lifetime is strictly bound to the player's UI session. It persists only as long as the portal summoning interface is visible to the player.

- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection under several conditions:
    - The player successfully summons the portal, which programmatically closes the page.
    - The player manually closes the UI screen.
    - The player disconnects from the server.
    - The underlying Portal Device block is destroyed.

## Internal State & Concurrency
- **State:** This class is stateful, but its core contextual state is immutable after construction. The constructor captures references to the player, the Portal Device block (`blockRef`), and the item being offered (`offeredItemStack`). This initial state is used for all subsequent validation and processing. The class does not contain mutable fields that change over its lifetime; instead, it re-computes the world state on demand via `computeState`.

- **Thread Safety:** **This class is not thread-safe and must only be accessed from its owning world's main server thread.** All its methods assume they are executed within the server's primary update loop. The portal creation logic initiated by `handleDataEvent` is highly asynchronous, leveraging `CompletableFuture` to avoid blocking the server thread. Callbacks and state mutations (like `worldChunk.setBlock`) are carefully marshaled back onto the appropriate world thread to prevent race conditions and ensure data consistency.

**WARNING:** Directly invoking methods on this class from other threads will lead to severe concurrency issues, including world corruption and server instability.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, commandBuilder, eventBuilder, store) | void | O(N) | Called by the UI framework to generate the UI commands based on the current game state. Complexity is proportional to the number of UI elements defined in the portal asset. |
| handleDataEvent(ref, store, data) | void | O(async) | Processes UI events from the client. Triggers the complex, asynchronous world-spawning pipeline when the "Summon" action is received. |

## Integration Patterns

### Standard Usage
This class is intended to be used by server-side systems that handle player interactions. The typical flow involves creating the page and immediately assigning it to the player.

```java
// In a block interaction handler...
Player player = ...;
Ref<ChunkStore> blockRef = ...;
ItemStack itemInHand = player.getInventory().getItemInHand();
PortalDeviceConfig config = ...;

// Create and display the UI page for the player
PortalDeviceSummonPage page = new PortalDeviceSummonPage(player.getRef(), config, blockRef, itemInHand);
player.getPageManager().setPage(player.getRef(), store, page);
```

### Anti-Patterns (Do NOT do this)
- **State Re-use:** Do not attempt to cache or re-use an instance of PortalDeviceSummonPage. It is cheap to create and is designed to capture the specific state of a single interaction attempt.

- **Direct Event Handling:** Never call `handleDataEvent` manually. This method is part of the contract with the server's UI framework and should only be invoked in response to a network packet from the client.

- **Blocking on Portal Creation:** Do not write code that assumes the portal and its world exist immediately after `handleDataEvent` is called. The creation process is asynchronous and may take several ticks or even fail. Use the `CompletableFuture` chain if you need to react to the outcome.

## Data Pipeline
The flow of data and control for a successful portal summoning operation is a round-trip process involving the client, server, and multiple backend systems.

> Flow:
> Player Interaction -> Server Event -> **new PortalDeviceSummonPage()** -> `build()` -> UI Commands to Client -> Client Renders UI -> Player Clicks Summon -> UI Event to Server -> **`handleDataEvent()`** -> `InstancesPlugin.spawnInstance()` -> Asynchronous World Creation -> `WorldChunk.setBlock()` -> Final State Change

