---
description: Architectural reference for InventoryHelper
---

# InventoryHelper

**Package:** com.hypixel.hytale.server.npc.util
**Type:** Utility

## Definition
```java
// Signature
public class InventoryHelper {
```

## Architecture & Concepts
The InventoryHelper is a stateless, static utility class designed to provide a high-level, simplified API for interacting with the server-side inventory system. It serves as a facade over the more granular components like Inventory, ItemContainer, and ItemStack, abstracting away the complexities of direct slot management, item lookups, and state validation.

Its primary role is to support the NPC and scripting systems by offering a robust, centralized set of tools for common inventory operations. By consolidating this logic, it ensures consistent behavior and reduces the likelihood of errors that can arise from direct, ad-hoc manipulation of inventory slots.

A key architectural feature is its reliance on glob-based string matching for identifying items (e.g., "hytale:wood_sword_*"). This provides significant flexibility for game designers and scripters, allowing them to target categories of items without needing to specify every single item ID. All operations are performed directly on the Inventory instances passed as arguments, making the helper a pure-logic layer with no internal state.

### Lifecycle & Ownership
- **Creation:** This class is never instantiated. Its private constructor prevents the creation of any instances, enforcing its role as a purely static utility. The class is loaded by the JVM ClassLoader at runtime when first referenced.
- **Scope:** Application-wide. As a static class, its methods are available globally across the server for the entire duration of the application's lifecycle.
- **Destruction:** The class is unloaded from memory only when the Java Virtual Machine shuts down.

## Internal State & Concurrency
- **State:** The InventoryHelper is completely **stateless**. It holds no mutable or immutable fields, either static or instance-based. All methods operate exclusively on the arguments provided by the caller.

- **Thread Safety:** While the InventoryHelper class itself is inherently thread-safe due to its stateless nature, the operations it performs are **not**. It directly mutates the state of Inventory and ItemContainer objects.

  **WARNING:** The caller is solely responsible for ensuring thread safety. If an Inventory object can be accessed by multiple threads (e.g., the main game thread and an AI behavior thread), all calls to InventoryHelper methods on that inventory **must** be synchronized using external locks. Failure to do so will result in severe race conditions, data corruption, and unpredictable inventory states.

## API Surface
The public API consists entirely of static methods designed for querying or mutating an Inventory.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesItem(pattern, itemStack) | boolean | O(k) | Core predicate. Checks if an ItemStack's ID matches a glob pattern. |
| findInventorySlotWithItem(inv, name) | short | O(N) | Scans the main inventory and hotbar to find the first slot containing a matching item. |
| useItem(inv, name, slotHint) | boolean | O(N) | High-level action to equip an item. Searches for the item, or creates it if not found, and sets it as the active item in hand. |
| useArmor(armorInv, itemStack) | boolean | O(1) | Places a given armor ItemStack into its designated armor slot. |
| countItems(container, names) | int | O(N) | Counts the total quantity of items matching a list of patterns within a container. |
| checkHotbarSlot(inv, slot) | boolean | O(1) | Validates if a given slot index is within the bounds of the inventory's hotbar. |

## Integration Patterns

### Standard Usage
The intended use is to call its static methods directly, passing in the target NPC's Inventory object. This is common within AI behavior nodes or scripted server events.

```java
// Example: An AI behavior node that ensures an NPC has a sword equipped.
Inventory npcInventory = npc.getInventory();
String swordPattern = "hytale:iron_sword";

// Check if the NPC is already holding a sword.
if (!InventoryHelper.holdsItem(npcInventory, swordPattern)) {
    // If not, find a sword in the inventory and equip it.
    // This will handle finding the item and setting the active hotbar slot.
    boolean success = InventoryHelper.useItem(npcInventory, swordPattern);
    if (!success) {
        // Handle the case where the NPC has no sword to equip.
        log("NPC " + npc.getId() + " has no sword to use.");
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Attempted Instantiation:** The class has a private constructor. Do not attempt to create an instance of InventoryHelper via reflection or other means. It is designed to be purely static.
- **Unsynchronized Concurrent Access:** The most critical anti-pattern. Never call InventoryHelper methods on the same Inventory object from multiple threads without an external locking mechanism. This will corrupt inventory data.
- **Ignoring Boolean Return Values:** Many mutator methods like `useItem` or `setHotbarItem` return a boolean to indicate success or failure. Code that ignores this return value may proceed under false assumptions about the state of the inventory.
- **Passing Invalid Item IDs:** Calling methods like `createItem` or `useItem` with an item ID string that does not correspond to a loaded game asset will result in failure or null objects. Always validate asset existence with `itemKeyExists` if the source is untrusted.

## Data Pipeline
InventoryHelper does not participate in a data pipeline in the traditional sense. Instead, it acts as a command processor that translates high-level intentions into low-level state changes within the Inventory system.

> Flow:
> AI Behavior Tree (e.g., "Attack Target") -> Issues Command ("Equip Sword") -> **InventoryHelper.useItem(inventory, "hytale:sword")** -> Mutates `Inventory` and `ItemContainer` state -> Server broadcasts inventory update -> Game World renders NPC holding the sword.

