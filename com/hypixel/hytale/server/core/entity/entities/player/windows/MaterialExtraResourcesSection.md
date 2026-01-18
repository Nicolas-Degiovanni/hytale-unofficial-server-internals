---
description: Architectural reference for MaterialExtraResourcesSection
---

# MaterialExtraResourcesSection

**Package:** com.hypixel.hytale.server.core.entity.entities.player.windows
**Type:** Transient

## Definition
```java
// Signature
public class MaterialExtraResourcesSection {
```

## Architecture & Concepts
The MaterialExtraResourcesSection class is a server-side data model that represents a dynamic component of a player's user interface, specifically one that displays required materials for an action like crafting or enchanting. It acts as a state container, bridging the server's core inventory logic with the client-server network protocol.

This class is not a persistent entity. It is a short-lived data structure created in response to a player's interaction with a game world object (e.g., a crafting table). Its primary responsibility is to aggregate the necessary material requirements, link them to a physical ItemContainer for validation, and serialize this state into a network packet for client-side rendering. It embodies the Model in a server-authoritative Model-View-Controller (MVC) architecture for player UI windows.

## Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level window or container management system on the server when a player opens a UI that requires a list of supplemental materials. It is never created directly by client-side code.
- **Scope:** The lifetime of a MaterialExtraResourcesSection instance is strictly bound to the player's current UI session. It exists only as long as the corresponding window is open and the specific interaction is active.
- **Destruction:** The object is eligible for garbage collection as soon as the player closes the UI window or completes the associated action. There are no explicit cleanup methods; ownership is relinquished by the parent window controller.

## Internal State & Concurrency
- **State:** Highly mutable. The internal state, including the list of materials and the validity flag, is designed to be modified frequently by the server's game logic as the player interacts with the UI (e.g., adding or removing items). It holds direct references to other mutable objects like ItemContainer.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization mechanisms and is designed to be accessed exclusively by the server's main game thread or a dedicated thread for a specific player session. Unsynchronized access from multiple threads will lead to race conditions and data corruption. All interactions must be marshaled onto the appropriate game thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setExtraMaterials(ItemQuantity[]) | void | O(1) | Overwrites the list of required materials. |
| isValid() | boolean | O(1) | Returns true if the current state of the section is considered valid for a transaction. |
| setValid(boolean) | void | O(1) | Manually sets the validity flag, typically after server-side validation. |
| toPacket() | ExtraResources | O(N) | Serializes the current state into a network packet for client synchronization. N is the number of materials. |
| getItemContainer() | ItemContainer | O(1) | Retrieves the associated server-side inventory container. |
| setItemContainer(ItemContainer) | void | O(1) | Binds this section to a specific server-side inventory container. |

## Integration Patterns

### Standard Usage
This object is managed by a parent controller. The controller populates its state based on game rules and then uses it to generate a network packet to update the client.

```java
// Server-side window management logic
void onPlayerOpenCraftingWindow(Player player, Recipe recipe) {
    // Create a new, transient section for this UI interaction
    MaterialExtraResourcesSection section = new MaterialExtraResourcesSection();

    // Populate from game data
    section.setExtraMaterials(recipe.getRequiredMaterials());
    section.setItemContainer(player.getInventory().getCraftingGrid());
    section.setValid(false); // Initially invalid until player adds items

    // Serialize to packet and send to client
    ExtraResources packet = section.toPacket();
    player.getConnection().sendPacket(packet);
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not reuse a MaterialExtraResourcesSection instance for a new UI interaction. Its state is specific to a single context and must be re-initialized from scratch.
- **Concurrent Modification:** Do not access or modify an instance from any thread other than the one that owns the player's game state. This will break server state guarantees.
- **Client-Side Instantiation:** This is a server-only data model. The client receives the serialized ExtraResources packet; it should never have a concept of the MaterialExtraResourcesSection class itself.

## Data Pipeline
The primary function of this class is to serve as a node in the server-to-client UI update pipeline.

> Flow:
> Server Game Event (e.g., Player opens Anvil) -> WindowManager creates **MaterialExtraResourcesSection** -> State is populated from Recipe & PlayerInventory -> **toPacket()** is called -> ExtraResources Packet -> Network Layer -> Client UI Framework -> UI Render Update

