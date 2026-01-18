---
description: Architectural reference for ItemStackItemContainer
---

# ItemStackItemContainer

**Package:** com.hypixel.hytale.server.core.inventory.container
**Type:** Transient

## Definition
```java
// Signature
public class ItemStackItemContainer extends ItemContainer {
```

## Architecture & Concepts

The ItemStackItemContainer is a specialized, virtual implementation of the ItemContainer contract. Its primary architectural purpose is to represent an inventory that is physically stored within the metadata of another single ItemStack. This pattern enables items like backpacks, toolbelts, or containers-within-containers, where one item in a parent inventory logically expands into its own grid of slots.

Unlike standard containers such as chests which manage a direct, server-owned inventory state, an ItemStackItemContainer acts as an **Adapter**. It adapts the BSON data stored in an ItemStack's metadata to the high-level ItemContainer interface. It does not own its state; it provides a transactional, in-memory view of the parent ItemStack's state.

All modifications performed on an ItemStackItemContainer, such as adding or removing items, are not stored within the container itself. Instead, these operations are immediately serialized back into the BSON metadata of the original ItemStack and written back into the parent container. This ensures that the container's state is intrinsically linked to the item that represents it. If the parent item is moved, dropped, or destroyed, its contained inventory moves with it.

The core mechanism relies on a set of static `Codec` definitions (CONTAINER_CODEC, CAPACITY_CODEC, ITEMS_CODEC) to define the BSON schema for the nested inventory data.

### Lifecycle & Ownership
-   **Creation:** Instances are never created directly via a public constructor. They are exclusively instantiated on-demand through static factory methods like `getContainer` or `ensureContainer`. These methods are called when the game logic needs to interact with the contents of a container-like item, for example, when a player opens their backpack. The factory method reads the parent ItemStack's metadata and constructs a transient container view.

-   **Scope:** The lifetime of an ItemStackItemContainer is intended to be short and transactional. It exists only for the duration of a specific interaction, such as a UI screen being open or an automated process transferring items. It is a temporary, stateful proxy to the underlying BSON data.

-   **Destruction:** The object is eligible for garbage collection as soon as it goes out of scope. There is no explicit destruction or cleanup method. The state it represents persists within the parent ItemStack's metadata, not in the container object itself.

## Internal State & Concurrency
-   **State:** The class is highly mutable. Its primary state consists of the `items` array, which is a deserialized, in-memory copy of the inventory data from the parent ItemStack's BSON. It also maintains immutable references to the `parentContainer`, the `itemStackSlot`, and the `originalItemStack` to maintain its link to the data source.

-   **Thread Safety:** This class is thread-safe. All read and write operations on the internal `items` array are guarded by a `ReentrantReadWriteLock`. The protected `readAction` and `writeAction` methods provide a robust locking mechanism for all internal data access, preventing race conditions when multiple systems might attempt to interact with the same container item simultaneously. Filter management uses concurrent map implementations for safe modification from multiple threads.

## API Surface

The primary public contract is defined by the static factory methods, which serve as the main entry points for all interactions.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getContainer(container, slot) | ItemStackItemContainer | O(N) | Attempts to create a container view from an existing ItemStack. Returns null if the item has no container metadata. N is the size of the BSON document. |
| makeContainerWithCapacity(...) | ItemStackItemContainer | O(N) | Initializes and attaches new container metadata to an ItemStack. Throws IllegalStateException if the item already has a container. |
| ensureContainer(container, slot, capacity) | ItemStackItemContainer | O(N) | Retrieves an existing container view or creates one if it does not exist. This is the primary idempotent factory method. |
| ensureConfiguredContainer(...) | ItemStackItemContainer | O(N+M) | Ensures a container exists and applies a set of predefined filters from an ItemStackContainerConfig. M is the capacity. |
| isItemStackValid() | boolean | O(1) | Critical validation check. Verifies that the item in the parent container's slot has not been replaced by a different item type. |
| writeToItemStack(...) | static void | O(N) | The core persistence mechanism. Serializes the `items` array into BSON and updates the metadata of the target ItemStack in its parent container. |

## Integration Patterns

### Standard Usage

An ItemStackItemContainer should always be fetched, used for a series of operations, and then discarded. Do not store long-lived references to it.

```java
// Assume 'playerInventory' is a valid ItemContainer and slot 5 holds a backpack
short backpackSlot = 5;
short backpackCapacity = 27;

// Get or create the container view for the backpack item
ItemStackItemContainer backpack = ItemStackItemContainer.ensureContainer(playerInventory, backpackSlot, backpackCapacity);

if (backpack != null) {
    // The backpack object can now be treated like any other container
    backpack.addItem(new ItemStack("stone", 64));
    ItemStack currentItem = backpack.getItemStack((short) 0);
    
    // All changes are automatically written back to the backpack ItemStack's metadata.
    // The 'backpack' variable can now be discarded.
}
```

### Anti-Patterns (Do NOT do this)
-   **Long-Lived References:** Storing an instance of ItemStackItemContainer in a component field is highly discouraged. The underlying ItemStack in the parent container can be moved, modified, or destroyed by other game systems. Always re-fetch the container using `getContainer` before an operation to ensure you are operating on valid, current data.

-   **Ignoring Validity Checks:** Failing to call `isItemStackValid` can lead to critical errors. If the original item (e.g., a backpack) is swapped with another item (e.g., a sword), the container instance becomes an orphaned, invalid view. Any subsequent operations will throw an `IllegalStateException`.

-   **External Modification of Parent Item:** Modifying the parent ItemStack's metadata via other means while an ItemStackItemContainer is active can lead to state desynchronization. The container's in-memory `items` array will be out of sync with the BSON data, and the next write from the container will overwrite any external changes.

## Data Pipeline

The flow of data through this component is bidirectional, acting as a bridge between the high-level inventory system and low-level item metadata.

> **Read Flow:**
> External System Call (`getContainer`) -> `ItemContainer.getItemStack(slot)` -> `ItemStack.getFromMetadataOrNull` -> BSON document is parsed by `CONTAINER_CODEC` -> `ITEMS_CODEC` decodes BSON array into `ItemStack[]` -> **New ItemStackItemContainer instance is created with the in-memory array**.

> **Write Flow:**
> External System Call (`addItem`, `setSlot`, etc.) -> Internal `items` array is mutated -> `writeToItemStack` is invoked -> `ITEMS_CODEC` encodes the in-memory `ItemStack[]` into a BSON array -> The new BSON is written to the parent ItemStack's metadata -> `ItemContainer.setItemStackForSlot` persists the modified ItemStack in the parent container.

