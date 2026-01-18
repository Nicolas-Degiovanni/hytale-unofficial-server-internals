---
description: Architectural reference for ItemStack
---

# ItemStack

**Package:** com.hypixel.hytale.server.core.inventory
**Type:** Transient Value Object

## Definition
```java
// Signature
public class ItemStack implements NetworkSerializable<ItemWithAllMetadata> {
```

## Architecture & Concepts

The ItemStack is the fundamental, immutable data structure representing a single stack of items within the game world. It serves as the canonical representation for items held in inventories, dropped on the ground, or involved in crafting and trade operations. It is not merely a reference to an item type, but a complete snapshot of a stack's state, including its quantity, durability, and any attached custom data.

Architecturally, ItemStack is designed as a pure data container with three primary responsibilities:
1.  **State Representation:** Encapsulates the core attributes of an item stack: `itemId`, `quantity`, `durability`, and `metadata`.
2.  **Immutability:** All modification operations (e.g., changing quantity, applying damage) do not alter the instance's state. Instead, they return a *new* ItemStack instance with the updated values. This design is critical for predictable state management and thread safety, preventing entire classes of bugs related to shared mutable state.
3.  **Serialization:** Through the `NetworkSerializable` interface and the static `CODEC` field, ItemStack defines its own contract for conversion to and from other data formats. The `toPacket` method handles conversion to a network-optimized protocol object, while the `CODEC` facilitates serialization to persistent formats like BSON for saving game state.

The static `CODEC` field is a cornerstone of its design, leveraging Hytale's declarative codec system. This codec defines the schema, validation rules (e.g., quantity must be positive), and data-binding logic for persistence, ensuring data integrity across system boundaries.

## Lifecycle & Ownership

-   **Creation:** ItemStack instances are ephemeral and created on-demand by various game systems. Common creation points include loot generation from entities, world generation for chests, player crafting results, or administrative commands. The constructors are public, allowing for direct instantiation where needed.
-   **Scope:** The lifetime of an ItemStack is tied to its container. It persists as long as it is referenced by an inventory slot, a world entity, or a temporary variable within a game logic function. It is not a long-lived, session-scoped service.
-   **Destruction:** As a standard Java object, an ItemStack is eligible for garbage collection as soon as all references to it are removed. For example, when a stack is fully consumed or moved, the original object is dereferenced and eventually reclaimed by the JVM.

## Internal State & Concurrency

-   **State:** The class is **Immutable**. Once an ItemStack is constructed, its core data (`itemId`, `quantity`, `durability`, `metadata`) cannot be changed. Methods that appear to modify the object, such as `withQuantity` or `withDurability`, return a new instance.
    -   A private, mutable `cachedPacket` field exists as a performance optimization to avoid redundant serialization.

-   **Thread Safety:** The immutable design makes ItemStack instances inherently **thread-safe for read operations**. Any number of threads can access an instance's data without risk of corruption or race conditions.
    -   **Warning:** The lazy initialization of `cachedPacket` in the `toPacket` method is not protected by a lock. In the unlikely scenario that multiple threads call `toPacket` simultaneously on a newly created instance, the packet object may be constructed more than once. This is a benign race condition that does not affect the integrity of the core item data.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getItem() | Item | O(1) | Resolves the `itemId` string into a concrete Item asset from the global asset map. |
| withQuantity(int quantity) | ItemStack | O(1) | Creates a new ItemStack with the specified quantity. Returns null if quantity is zero. |
| withDurability(double durability) | ItemStack | O(1) | Creates a new ItemStack with the specified durability, clamped between 0 and max. |
| withMetadata(BsonDocument meta) | ItemStack | O(1) | Creates a new ItemStack with the provided BSON metadata document. |
| isStackableWith(ItemStack other) | boolean | O(N) | Checks if this stack can be merged with another. Complexity depends on metadata size. |
| toPacket() | ItemWithAllMetadata | O(1) | Converts the object into its network packet representation. Caches the result. |
| getFromMetadataOrNull(KeyedCodec) | T | O(1) | Safely decodes a value from the internal BSON metadata using a typed codec. |

## Integration Patterns

### Standard Usage

The correct pattern for modifying an ItemStack is to replace the old reference with the new instance returned by a `with...` method. This respects its immutable contract.

```java
// How a developer should normally use this
ItemStack sword = new ItemStack("hytale:iron_sword", 1);

// To apply damage, create a new stack and replace the old one.
ItemStack damagedSword = sword.withIncreasedDurability(-10.0);

// inventory.setSlot(0, damagedSword);
```

### Anti-Patterns (Do NOT do this)

-   **Assuming Mutability:** Do not call a `with...` method and expect the original object to change. This is a common source of bugs.
    ```java
    // BAD: This has no effect. The returned object is discarded.
    sword.withIncreasedDurability(-10.0); 
    ```
-   **Manual Null/Empty Checks:** Avoid manual null checks. Use the provided static utility method which is safer and more expressive.
    ```java
    // BAD
    if (stack == null || stack.isEmpty()) { ... }

    // GOOD
    if (ItemStack.isEmpty(stack)) { ... }
    ```
-   **Modifying Metadata Directly:** The deprecated `getMetadata` method returns a clone. Modifying this clone has no effect on the original ItemStack. Use the `withMetadata` methods to apply changes.

## Data Pipeline

ItemStack is a critical node in the data flow between game logic, the network, and persistent storage.

> **Flow 1: Game State to Network Packet**
>
> Game Logic -> Creates/modifies **ItemStack** -> `toPacket()` -> `ItemWithAllMetadata` -> Network Encoder -> Client

> **Flow 2: Persistent Storage to Game State**
>
> Database (BSON) -> `ItemStack.CODEC.decode()` -> **ItemStack** -> Inventory System -> Game Logic

---

