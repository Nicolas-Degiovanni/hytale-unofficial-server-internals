---
description: Architectural reference for SimpleItemContainer
---

# SimpleItemContainer

**Package:** com.hypixel.hytale.server.core.inventory.container
**Type:** Stateful Component

## Definition
```java
// Signature
public class SimpleItemContainer extends ItemContainer {
```

## Architecture & Concepts
The SimpleItemContainer is the server-side, canonical implementation of a generic inventory. It serves as the fundamental data structure for any entity or block that needs to store a collection of items, such as player inventories, chests, and crafting tables. Its design is centered on three critical architectural pillars: thread safety, data persistence, and transactional integrity.

At its core, the container manages a sparse map of slot indices to ItemStack objects. This is a deliberate choice to optimize for memory usage, as empty slots do not consume resources.

The most significant architectural feature is its built-in concurrency control. All read and write operations on the internal item map are strictly managed by a ReentrantReadWriteLock. This design makes the SimpleItemContainer inherently thread-safe, allowing it to be safely accessed and modified by different server systems simultaneously—for example, a player interacting with a chest (network thread) while an automated hopper attempts to insert an item (game loop thread).

Furthermore, the class includes a static BuilderCodec instance, making it fully serializable. This integration with the engine's codec system is crucial for world saving, chunk loading, and network synchronization. The codec handles the translation of the container's in-memory state to a persistent data format and back.

Finally, operations are designed to be transactional. Methods like addItemStack do not simply modify state; they return detailed transaction objects. These objects report the outcome, including any items that could not be added (the remainder), allowing calling systems to react robustly to partial successes or failures.

## Lifecycle & Ownership
- **Creation:** A SimpleItemContainer is instantiated when an entity requiring inventory capabilities is created. This can occur through direct instantiation (e.g., `new SimpleItemContainer(capacity)`) or, more commonly, through deserialization via its `CODEC` when a chunk or entity is loaded from storage. The static factory method `getNewContainer` is the preferred mechanism for programmatic creation.

- **Scope:** The lifecycle of a SimpleItemContainer is strictly bound to its owner. For a player entity, the container persists for the entire game session. For a world object like a chest, it exists as long as that block remains in the world.

- **Destruction:** The object is eligible for garbage collection when its owning entity is destroyed and all references to it are released. There is no explicit `destroy` or `close` method; cleanup is managed by the Java Virtual Machine.

## Internal State & Concurrency
- **State:** The component is highly mutable. Its primary state consists of the `items` map, which stores the ItemStacks, and the `capacity` field. It also maintains state for filtering rules via `globalFilter` and `slotFilters`.

- **Thread Safety:** This class is explicitly designed to be thread-safe. All internal data access is mediated by a `ReentrantReadWriteLock`.
    - **Read Operations:** Methods like `getItemStack` and `isEmpty` acquire a read lock, allowing for concurrent reads from multiple threads.
    - **Write Operations:** Methods that modify the inventory, such as `internal_setSlot` and `internal_clear`, acquire an exclusive write lock, preventing all other read and write operations until the modification is complete.
    - **WARNING:** Any extension of this class must rigorously adhere to this locking pattern. Failure to acquire the appropriate lock before accessing the `items` map will break thread safety guarantees and lead to severe data corruption.

## API Surface
The public API provides a transactional and thread-safe contract for inventory manipulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCapacity() | short | O(1) | Returns the maximum number of slots in the container. |
| getItemStack(slot) | ItemStack | O(1) | Safely retrieves the ItemStack in a given slot. Returns null if the slot is empty. |
| addItemStack(itemStack) | ItemStackTransaction | O(N) | Attempts to add an ItemStack to any available slot. Returns a transaction result detailing the outcome and any remaining items. |
| addItemStackToSlot(slot, itemStack) | ItemStackSlotTransaction | O(1) | Attempts to add an ItemStack to a specific slot. Returns a transaction result. |
| setGlobalFilter(filter) | void | O(1) | Applies a container-wide filter to control which items can be added or removed. |
| setSlotFilter(actionType, slot, filter) | void | O(1) | Applies a fine-grained filter to a specific slot for a specific action (e.g., ADD, REMOVE). |
| clone() | SimpleItemContainer | O(N) | Creates a deep copy of the container, including a snapshot of its items. |

## Integration Patterns

### Standard Usage
The most common pattern involves using the static helper methods to perform a complete "add-or-drop" operation. This pattern ensures that if an item cannot be placed in an inventory, it is correctly spawned in the world as a dropped entity.

```java
// High-level game logic for giving an item to a player or entity.
// store: A reference to the world's entity storage system.
// ref: A reference to the entity that owns the container.
// itemContainer: The target inventory.
// itemStack: The item to be added.

// This single method call handles the entire transaction.
SimpleItemContainer.addOrDropItemStack(store, ref, itemContainer, itemStack);
```

### Anti-Patterns (Do NOT do this)
- **Unsafe Iteration:** Do not attempt to iterate over the container's contents without using the provided thread-safe API. Directly accessing the internal `items` map from a subclass without acquiring a lock will cause unpredictable behavior and crashes.

- **External State Mutation:** Do not retrieve an ItemStack via `getItemStack` and then modify its state directly. This bypasses the container's transactional logic and filtering rules. All modifications should be performed through methods like `addItemStack` or `setSlot`.
    - **BAD:** `ItemStack stack = container.getItemStack(0); stack.setCount(10);`
    - **GOOD:** `container.setSlot(0, newStack);`

- **Prolonged Locking:** If extending this class, avoid performing long-running or blocking operations (e.g., network calls, file I/O) while holding a write lock. This will cause significant performance degradation and can lead to server-wide deadlocks.

## Data Pipeline
The SimpleItemContainer is a critical component in the flow of item data during gameplay. A typical user interaction follows this pipeline:

> Flow:
> Client Input Packet (e.g., "move item") -> Network Thread -> Server-side Player Logic -> **SimpleItemContainer.addItemStackToSlot()** -> Internal Lock Acquired -> State Modified -> Transaction Result -> Server Logic (sends response packet) -> Client UI Update

