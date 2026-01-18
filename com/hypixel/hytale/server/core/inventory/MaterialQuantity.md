---
description: Architectural reference for MaterialQuantity
---

# MaterialQuantity

**Package:** com.hypixel.hytale.server.core.inventory
**Type:** Transient / Value Object

## Definition
```java
// Signature
public class MaterialQuantity implements NetworkSerializable<com.hypixel.hytale.protocol.MaterialQuantity> {
```

## Architecture & Concepts
MaterialQuantity is a fundamental data structure that represents a specific quantity of a game material. It is not a service or a managed entity, but rather a lightweight Value Object used throughout the server to define costs, rewards, and inventory contents.

Architecturally, it serves three primary functions:
1.  **Data Definition:** Through its static **CODEC**, it provides a schema for deserializing material definitions from configuration files (e.g., JSON or BSON). This makes it a cornerstone of systems like crafting, block drops, and loot tables.
2.  **Abstract Representation:** It acts as a generalized container that can represent a concrete item (via **itemId**), a raw resource (via **resourceTypeId**), or a category of items (via **tag**). This flexibility allows game systems to operate on materials without needing to know the specific underlying type.
3.  **Network Serialization:** By implementing the NetworkSerializable interface, it defines the contract for its own transmission between the server and client. This is critical for synchronizing data like recipe books or container contents.

The presence of a **tagIndex** field is a key performance optimization. During the deserialization lifecycle via the **CODEC**, the string-based **tag** is resolved to an integer index by the AssetRegistry. Subsequent lookups and comparisons can then use fast integer operations instead of more expensive string comparisons.

## Lifecycle & Ownership
-   **Creation:** MaterialQuantity instances are created in two primary ways:
    1.  **Deserialization:** The most common path. The server's data loading systems use the static **CODEC** to parse configuration files and instantiate MaterialQuantity objects as part of larger definitions like recipes or loot tables.
    2.  **Programmatic Instantiation:** Game logic may create instances directly using the constructor (e.g., `new MaterialQuantity(...)`) to represent a dynamic material requirement or to perform an inventory operation.
-   **Scope:** Transient and short-lived. An instance typically exists only for the duration of a single operation, such as a crafting check, an inventory transaction, or the construction of a network packet. They are not designed to be long-lived, stateful objects.
-   **Destruction:** Instances are managed by the Java garbage collector. They are eligible for cleanup as soon as they are no longer referenced. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
-   **State:** Mutable. The internal fields are not final. However, it is designed to be treated as an immutable value object after its initial creation. The primary mutation occurs during the deserialization process managed by its **CODEC**. The **tagIndex** is a derived, cached value populated in the `afterDecode` lifecycle hook.
-   **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization mechanisms. It must be confined to a single thread, typically the main server game loop. Passing instances between threads or modifying them concurrently will result in undefined behavior and data corruption.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| clone(int quantity) | MaterialQuantity | O(1) | Creates a shallow copy of the object with a new quantity. The underlying metadata is not deep-copied. |
| toItemStack() | ItemStack | O(1) | Converts the object to an ItemStack. Returns null if **itemId** is not present. |
| toResource() | ResourceQuantity | O(1) | Converts the object to a ResourceQuantity. Returns null if **resourceTypeId** is not present. |
| toPacket() | com.hypixel.hytale.protocol.MaterialQuantity | O(1) | Serializes the object into its network-ready protocol representation for client-server communication. |

## Integration Patterns

### Standard Usage
MaterialQuantity is most often encountered as a deserialized component of a larger data structure, such as a crafting recipe. Game logic then uses it to validate crafting inputs or create output stacks.

```java
// Example: A system processing a crafting recipe
// The 'recipe' object would have been loaded from a config file using the Codec system.
Recipe recipe = getRecipeFromRegistry("iron_pickaxe");
MaterialQuantity[] costs = recipe.getCosts();

// Check if a player's inventory can satisfy the cost
if (playerInventory.hasMaterials(costs)) {
    playerInventory.removeMaterials(costs);
    ItemStack result = recipe.getOutput().toItemStack();
    playerInventory.addItem(result);
}
```

### Anti-Patterns (Do NOT do this)
-   **Post-Creation Mutation:** Do not modify the fields of a MaterialQuantity after it has been passed into a game system. If you need a variant with a different quantity, use the **clone** method. Treating it as immutable prevents complex side effects.
-   **Concurrent Access:** Never share and modify a MaterialQuantity instance across multiple threads. It must be confined to a single thread's scope of execution.
-   **Assuming Identifier Presence:** Do not assume that **itemId**, **resourceTypeId**, and **tag** are all populated. Logic must be written to handle cases where any of these are null, as this is valid by design.

## Data Pipeline
MaterialQuantity is a critical link in both the server's data definition pipeline and the network synchronization pipeline.

> **Data Definition Flow:**
> BSON/JSON Config File -> Server Asset Loader -> **BuilderCodec** -> **MaterialQuantity** instance -> In-memory Recipe/LootTable

> **Network Synchronization Flow:**
> Server-side Game Event -> **MaterialQuantity** instance created -> **toPacket()** method called -> Network Packet -> Client -> Deserialization -> Client-side UI Update

