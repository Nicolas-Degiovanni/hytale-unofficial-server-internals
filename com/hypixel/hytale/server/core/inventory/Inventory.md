---
description: Architectural reference for Inventory
---

# Inventory

**Package:** com.hypixel.hytale.server.core.inventory
**Type:** Transient

## Definition
```java
// Signature
public class Inventory implements NetworkSerializable<UpdatePlayerInventory> {
```

## Architecture & Concepts
The Inventory class is a high-level state management component responsible for a **LivingEntity**'s entire collection of items. It is not a single container, but rather an aggregate facade over multiple specialized **ItemContainer** instances, each representing a distinct section: **storage**, **hotbar**, **armor**, **utility**, and **backpack**.

Architecturally, it serves three primary functions:
1.  **State Aggregation:** It unifies various item storage sections into a single, coherent API. This simplifies item management for other game systems, which do not need to be aware of the underlying container structure.
2.  **Transaction and Logic Hub:** It orchestrates complex item movements, such as **smartMoveItem**, which involves logic for equipping armor, merging stacks, or routing items to preferred containers. It also integrates with the **InteractionManager** to handle gameplay-sensitive operations like swapping an item into an active hand slot, which may trigger game interactions.
3.  **Change Notification and Synchronization:** It is the primary source of truth for an entity's items. Through event listeners on its underlying containers, it propagates change events (**LivingEntityInventoryChangeEvent**) to the server's **EventBus**. It also manages dirty flags (**isDirty**, **needsSaving**) to signal the networking and persistence layers when state synchronization is required.

A key internal concept is the use of **CombinedItemContainer**. These are virtual, read-only views that chain multiple physical containers together. This is a performance optimization that allows operations like searching for an item or adding a picked-up item to query across the hotbar and main storage without iterating over each one separately or merging them into a temporary collection.

## Lifecycle & Ownership
-   **Creation:** An Inventory object is instantiated as a component of a **LivingEntity**, typically a **Player**. This occurs either during the entity's initial creation (via the default constructor) or through deserialization from persistent storage using its associated **BuilderCodec**. It is fundamentally owned by its parent entity.
-   **Scope:** The instance's lifetime is strictly bound to its owning **LivingEntity**. It persists as long as the entity is loaded in the world.
-   **Destruction:** The **unregister** method is the critical cleanup hook. It must be called when the owning entity is unloaded or destroyed. This method detaches all event listeners from the underlying **ItemContainer**s, preventing severe memory leaks where the Inventory object would be retained by the global **EventBus** after its owner is gone.

## Internal State & Concurrency
-   **State:** The Inventory class is highly mutable. Its core state consists of references to multiple **ItemContainer** objects, which are themselves mutable collections. It also maintains state for the active slot index for the hotbar, utility, and tools sections. Two **AtomicBoolean** flags, **isDirty** and **needsSaving**, track the state for network synchronization and database persistence, respectively.
-   **Thread Safety:** This class is **not thread-safe** for mutations. All operations that modify item stacks (e.g., **moveItem**, **smartMoveItem**, **clear**) must be executed on the main server game thread. While the dirty flags use **AtomicBoolean**, this is only to allow safe flag-setting from other threads (e.g., a network thread acknowledging a packet). The core collections are not protected by locks, and concurrent modification will lead to data corruption and unpredictable behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| moveItem(fromSection, fromSlot, qty, toSection, toSlot) | void | O(1) | Executes a direct item move between two sections. **Warning:** This can trigger a complex **InteractionChain** if moving to/from an active hotbar slot. |
| smartMoveItem(fromSection, fromSlot, qty, moveType) | void | O(N) | Performs a context-aware move, attempting to equip armor, merge with existing stacks, or place in a preferred container based on the **SmartMoveType**. |
| setActiveSlot(sectionId, slot) | void | O(1) | Sets the currently active item slot for a given section (hotbar, utility). Triggers stat recalculation and invalidates the entity's equipment state for network updates. |
| getSectionById(id) | ItemContainer | O(1) | Retrieves a specific underlying container (e.g., hotbar, storage) by its constant section ID. Returns null for an invalid ID. |
| getContainerForItemPickup(item, settings) | ItemContainer | O(1) | Returns the appropriate virtual **CombinedItemContainer** to target for an item pickup, based on item type and player settings. |
| unregister() | void | O(1) | Critical cleanup method. Deregisters all internal event listeners to prevent memory leaks. Must be called on entity destruction. |
| toPacket() | UpdatePlayerInventory | O(N) | Serializes the entire inventory state into a network packet for client synchronization. |

## Integration Patterns

### Standard Usage
The Inventory is almost always accessed via its owning **LivingEntity**. Logic should retrieve the entity, get its inventory component, and then perform operations.

```java
// How a developer should normally use this
Player player = world.getPlayerByName("Player1");
if (player != null) {
    Inventory playerInventory = player.getInventory();
    // Example: Move 1 item from storage slot 0 to hotbar slot 1
    playerInventory.moveItem(Inventory.STORAGE_SECTION_ID, 0, 1, Inventory.HOTBAR_SECTION_ID, 1);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use **new Inventory()** to manage entity items. The inventory is part of the entity's component data and is managed by the entity lifecycle and persistence systems.
-   **Off-Thread Modification:** Never call methods like **moveItem** or access the underlying **ItemContainer**s from any thread other than the main server thread. This will cause race conditions.
-   **Forgetting to Unregister:** Failure to call **unregister** when an entity is removed from the world will cause a memory leak. The entity's inventory will not be garbage collected because its event listeners are still registered with the global event bus.

## Data Pipeline
The Inventory acts as a central node in several data flows.

> **Flow 1: Client Action to Inventory Change**
> Client Input -> C2S Network Packet (e.g., MoveItem) -> Packet Handler -> **Inventory.moveItem()** -> ItemContainer Change -> **Inventory** listener fires **LivingEntityInventoryChangeEvent** -> Event Bus

> **Flow 2: Inventory Change to Client Update**
> Game Logic (e.g., item pickup) -> **Inventory.getCombinedHotbarFirst().addItemStack()** -> ItemContainer Change -> **Inventory.markChanged()** is called -> Server Network Tick checks **consumeIsDirty()** -> **Inventory.toPacket()** -> S2C **UpdatePlayerInventory** Packet -> Client UI Update

> **Flow 3: Equipment Change to Stat Calculation**
> **Inventory.setActiveSlot()** OR Armor slot change -> **entity.invalidateEquipmentNetwork()** AND **statModifiersManager.setRecalculate(true)** -> Entity Stat System re-evaluates bonuses from equipped items.

