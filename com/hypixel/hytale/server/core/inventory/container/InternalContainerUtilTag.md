---
description: Architectural reference for InternalContainerUtilTag
---

# InternalContainerUtilTag

**Package:** com.hypixel.hytale.server.core.inventory.container
**Type:** Utility

## Definition
```java
// Signature
public class InternalContainerUtilTag {
```

## Architecture & Concepts

InternalContainerUtilTag is a stateless, package-private utility class designed to encapsulate the complex logic of manipulating items within an ItemContainer based on shared tags. Unlike direct item manipulation which targets a specific item definition, this class operates on abstract categories defined by a tag index (e.g., "all wood logs", "any ore", "edible fruit").

This class serves as a strategic helper, offloading a specialized and non-trivial piece of inventory logic from the core ItemContainer. Its methods are not intended for general consumption; they form an internal API used by the inventory system itself. The `protected` access level is a deliberate design choice to enforce this boundary.

A key architectural pattern is the separation of "test" operations from "mutation" operations. Methods prefixed with *test* allow the system to simulate a removal to determine if a requested quantity of a tagged item can be fulfilled. This pre-flight check is essential for transactional integrity, especially for features like crafting or trading where atomicity is required.

All mutation operations return detailed transaction objects (TagSlotTransaction, TagTransaction) instead of simple booleans. This provides the calling system with a rich, structured record of the changes, including the original state, the new state, the items moved, and any remaining quantities. This transactional data is critical for network synchronization, logging, and further game logic processing.

### Lifecycle & Ownership
- **Creation:** As a static utility class, InternalContainerUtilTag is never instantiated. Its methods are invoked directly on the class.
- **Scope:** The class and its static methods are loaded by the JVM's class loader and persist for the entire lifetime of the server application.
- **Destruction:** The class is unloaded when the server shuts down and its class loader is garbage collected.

## Internal State & Concurrency
- **State:** **Stateless and Immutable**. This class holds no member variables and has no internal state. All required data is passed as arguments to its static methods, primarily the target ItemContainer instance.

- **Thread Safety:** The class itself is inherently thread-safe. However, the operations it performs on the provided ItemContainer are stateful and **not** inherently safe. Concurrency control is entirely **delegated** to the ItemContainer instance. Every mutation method immediately calls `itemContainer.writeAction`, which secures a write lock on the container for the duration of the operation. This ensures that all modifications are atomic and isolated from other threads.

**WARNING:** Callers must never attempt to manage locking externally when using this utility. Doing so circumvents the container's own concurrency model and will lead to deadlocks or data corruption.

## API Surface

The API is `protected` and intended for use only by classes within the same package.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| internal_removeTagFromSlot(...) | TagSlotTransaction | O(1) | Attempts to remove a quantity of items matching a tag from a **single, specific slot**. |
| internal_removeTag(...) | TagTransaction | O(N) | Attempts to remove a quantity of items matching a tag from the **entire container**, iterating through slots until the request is fulfilled. |
| testRemoveTagFromItems(...) | int | O(N) | **Simulates** a tag removal across the entire container. Returns the quantity that could *not* be removed. Does not modify state. |
| testRemoveTagSlotFromItems(...) | TestRemoveItemSlotResult | O(N) | **Simulates** a tag removal across the entire container. Returns a structured result object. Does not modify state. |
| testRemoveTagFromSlot(...) | int | O(1) | **Simulates** a tag removal from a single, specific slot. Returns the quantity that could *not* be removed. Does not modify state. |

## Integration Patterns

### Standard Usage

This class is not meant to be used directly by feature developers. Its intended use is as a delegate for methods within the ItemContainer class itself.

```java
// Example of how ItemContainer might use this utility internally
// THIS CODE WOULD RESIDE INSIDE THE ItemContainer CLASS

public TagTransaction removeItemsByTag(int tagIndex, int quantity) {
    // The container delegates the complex, tag-based removal logic
    // to the specialized utility, passing itself as the target.
    return InternalContainerUtilTag.internal_removeTag(this, tagIndex, quantity, false, false, false);
}
```

### Anti-Patterns (Do NOT do this)
- **Public Invocation:** Do not attempt to make these methods public or call them from outside the `com.hypixel.hytale.server.core.inventory.container` package. They are an internal implementation detail of the inventory system.
- **Operating on Unlocked Containers:** Do not pass an ItemContainer to this utility without ensuring the container's own locking mechanism (`writeAction`) is used. The current implementation correctly wraps all logic in `writeAction`, and this pattern must be maintained.
- **Ignoring Transaction Results:** The returned transaction objects are not optional. They contain the ground truth of what occurred during the operation. Code that discards the result and assumes success will become a source of item duplication or loss bugs.

## Data Pipeline

InternalContainerUtilTag functions as a processing node in the inventory control flow. It does not source or sink data but rather transforms the state of an ItemContainer based on a set of rules.

> Flow:
> `ItemContainer.removeItemsByTag` call -> **`InternalContainerUtilTag.internal_removeTag`** -> `ItemContainer.writeAction` (acquires lock) -> Iterates and modifies `ItemContainer` internal slots -> `TagTransaction` result created -> Lock released -> `TagTransaction` returned to original caller.

