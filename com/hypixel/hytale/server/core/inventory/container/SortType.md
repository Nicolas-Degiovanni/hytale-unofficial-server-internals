---
description: Architectural reference for SortType
---

# SortType

**Package:** com.hypixel.hytale.server.core.inventory.container
**Type:** Utility

## Definition
```java
// Signature
public enum SortType {
```

## Architecture & Concepts
The SortType enum implements a **Strategy Pattern** to define and encapsulate different algorithms for sorting collections of ItemStack. Each enum constant (NAME, TYPE, RARITY) represents a distinct sorting strategy, providing a type-safe and extensible way to manage inventory sorting logic.

This class acts as a critical bridge between the server's inventory management system and the network protocol layer. It is responsible for translating a client's request to sort an inventory into a concrete, executable sorting operation on the server. The logic is self-contained within each enum constant's constructor, where a `java.util.Comparator` is built and configured.

Key architectural features include:
- **Compound Comparators:** The system supports multi-level sorting. For example, sorting by TYPE or RARITY will use NAME as a secondary, tie-breaking criterion. This is achieved by chaining comparators using `thenComparing`.
- **Inversion Control:** The `inverted` constructor parameter allows strategies like RARITY to sort in descending order (highest to lowest) while others sort in ascending order.
- **Protocol Decoupling:** The `toPacket` and `fromPacket` methods provide a clean abstraction layer, isolating the core server logic from the specific integer or enum values used in the network protocol. This allows the network format to change without affecting the inventory system.

## Lifecycle & Ownership
- **Creation:** As a Java enum, all instances (NAME, TYPE, RARITY) are instantiated by the JVM during class loading. They are compile-time constants.
- **Scope:** The instances are static and persist for the entire lifetime of the server application. They are globally accessible singletons.
- **Destruction:** The instances are garbage collected only when the server's class loader is unloaded, which typically happens during a full server shutdown.

## Internal State & Concurrency
- **State:** SortType is **Immutable**. Each enum constant holds a final reference to a `Comparator` instance. This comparator is configured once during static initialization and never changes.
- **Thread Safety:** This class is inherently thread-safe. Since its state is immutable and established before any application threads are active, its methods can be safely called from any thread without external synchronization. The returned comparators are also thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComparator() | Comparator | O(1) | Returns the pre-built comparator for this sorting strategy. |
| toPacket() | protocol.SortType | O(1) | Converts the server-side enum into its network protocol representation. |
| fromPacket(sortType) | SortType | O(1) | A static factory method to resolve a SortType from its network protocol representation. |

## Integration Patterns

### Standard Usage
The primary use case is to retrieve a comparator based on a client request and apply it to a list of ItemStacks within a container.

```java
// Example: Sorting a player's inventory based on a network request
com.hypixel.hytale.protocol.SortType requestedSort = packet.getSortType();
SortType serverSortType = SortType.fromPacket(requestedSort);

List<ItemStack> items = playerInventory.getItems();
items.sort(serverSortType.getComparator());

// The 'items' list is now sorted in place.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Enums cannot be instantiated with `new`. Attempting to do so will result in a compile-time error.
- **Relying on ordinal():** Do not use the `ordinal()` method for serialization or conditional logic. The order of enum declarations can change, leading to critical bugs. Always use the explicit `toPacket` and `fromPacket` methods for network communication.
- **Modifying VALUES:** The public static `VALUES` array should be treated as read-only. Modifying its contents at runtime will lead to unpredictable behavior across the entire server.

## Data Pipeline
SortType does not process a stream of data itself, but rather provides the logic used within the inventory management data pipeline.

> Flow:
> Client Sort Request Packet -> Network Handler -> `SortType.fromPacket()` -> **SortType** instance -> `getComparator()` -> Inventory System sorts items -> Inventory Update Packet -> Client UI reflects sorted items

