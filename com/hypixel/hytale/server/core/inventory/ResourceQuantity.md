---
description: Architectural reference for ResourceQuantity
---

# ResourceQuantity

**Package:** com.hypixel.hytale.server.core.inventory
**Type:** Transient / Value Object

## Definition
```java
// Signature
public class ResourceQuantity {
```

## Architecture & Concepts

The ResourceQuantity class is a fundamental data structure within the server's inventory and item management systems. It serves as a lightweight, immutable container that represents a specific quantity of a single type of resource, such as a stack of items.

Its core design principle is to decouple the *instance* of an item stack from the static *definition* of the item itself. This is achieved by using a string-based resourceId as a key. This key can be resolved against a full Item definition object at a later time to retrieve more complex properties, suchas its ItemResourceType.

This class is intentionally simple, containing only the identifier and the count. It is designed to be passed by value and frequently created or discarded during inventory operations like crafting, trading, or loot distribution. Its immutability is a key feature, ensuring that operations on an item stack (like splitting it) produce a new ResourceQuantity instance rather than modifying an existing one, which prevents a wide range of state corruption bugs.

## Lifecycle & Ownership

-   **Creation:** A ResourceQuantity is instantiated whenever a new item stack is required. This occurs during world generation (e.g., chest loot), when a player performs an action (e.g., mining a block, crafting an item), or when an existing stack is modified. The public constructor enforces data integrity by rejecting null resource identifiers and non-positive quantities.
-   **Scope:** The object's lifetime is typically short and bound to the scope of an inventory transaction or the lifecycle of its containing object, such as an ItemContainer. They are created, passed to methods, and then become eligible for garbage collection.
-   **Destruction:** As a simple Plain Old Java Object (POJO), ResourceQuantity is managed by the Java garbage collector. There are no native resources or explicit cleanup methods required. It is destroyed once it is no longer referenced.

## Internal State & Concurrency

-   **State:** Effectively immutable. While its fields are not declared final, there are no public or protected methods to modify the internal resourceId or quantity after construction. Any logical modification, such as changing the stack size, is achieved by creating a new instance via the clone method.
-   **Thread Safety:** This class is inherently thread-safe. Its immutable nature guarantees that multiple threads can read from an instance concurrently without locks or synchronization primitives. This makes it safe to use in multi-threaded contexts, such as passing item data between the main game loop and networking threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ResourceQuantity(resourceId, quantity) | constructor | O(1) | Creates a new instance. Throws IllegalArgumentException if quantity is not positive or NullPointerException if resourceId is null. |
| getResourceId() | String | O(1) | Returns the unique string identifier for the resource. |
| getQuantity() | int | O(1) | Returns the number of items in the stack. |
| clone(quantity) | ResourceQuantity | O(1) | Creates a new ResourceQuantity with the same resourceId but a new quantity. This is the primary mechanism for "modifying" a stack. |
| getResourceType(item) | ItemResourceType | O(N) | Searches the provided Item definition to find a matching resource type for this instance's resourceId. Complexity is linear to the number of resource types on the item. |

## Integration Patterns

### Standard Usage

ResourceQuantity should be treated as a simple data carrier. It is typically created to represent a change or a state within an inventory system.

```java
// Example: Representing a stack of 12 "hytale:wood_plank" items
ResourceQuantity planks = new ResourceQuantity("hytale:wood_plank", 12);

// Example: Splitting the stack to create a new, smaller stack
// The original 'planks' object remains unchanged.
ResourceQuantity smallerStack = planks.clone(4);

// Now, 'planks' still has a quantity of 12, and 'smallerStack' has a quantity of 4.
// The inventory system would then be responsible for reducing the original stack's quantity.
```

### Anti-Patterns (Do NOT do this)

-   **State Modification:** Do not attempt to use reflection to modify the internal quantity or resourceId fields. The entire inventory system relies on the immutability of this object for safe state transitions. Modifying an instance directly will lead to unpredictable and difficult-to-trace bugs.
-   **Invalid Instantiation:** Do not create instances with a quantity of zero or less. The constructor will throw an exception, and bypassing this check violates the object's contract. Empty item slots should be represented by a null reference, not a ResourceQuantity with a zero count.
-   **Misusing the Protected Constructor:** The protected no-argument constructor is reserved for framework or subclass use. Do not invoke it directly, as it results in an uninitialized and invalid object.

## Data Pipeline

ResourceQuantity does not process data itself; it *is* the data. It acts as a payload that flows between different systems.

> **Flow:**
>
> Loot Table Service -> **new ResourceQuantity("hytale:diamond", 1)** -> InventoryManager.addItemToContainer(container, resource) -> Container State Update -> Network Serialization -> Client-side Inventory Update

