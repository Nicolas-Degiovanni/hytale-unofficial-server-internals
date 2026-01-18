---
description: Architectural reference for InternalContainerUtilMaterial
---

# InternalContainerUtilMaterial

**Package:** com.hypixel.hytale.server.core.inventory.container
**Type:** Utility

## Definition
```java
// Signature
public class InternalContainerUtilMaterial {
```

## Architecture & Concepts

InternalContainerUtilMaterial is a stateless, internal utility class that forms a critical part of the server-side inventory transaction system. It functions as a **strategy dispatcher** for operations involving the `MaterialQuantity` abstraction.

The Hytale inventory system uses `MaterialQuantity` as a polymorphic representation of any removable game substance, which can be a concrete `ItemStack`, an abstract `Tag`, or a generic `Resource`. This class is responsible for introspecting a given `MaterialQuantity` object and delegating the requested operation to the correct, specialized utility class:

*   `InternalContainerUtilItemStack` for item-based materials.
*   `InternalContainerUtilTag` for tag-based materials.
*   `InternalContainerUtilResource` for resource-based materials.

The methods are prefixed with `internal_` or `test`, indicating they are not intended for direct use by high-level game logic. Instead, they provide the foundational implementation logic consumed by the `ItemContainer` class itself to ensure transactional integrity and consistent behavior across all material types. The `test` methods provide a "dry-run" or simulation capability, allowing the system to verify if an operation can succeed before committing any state changes.

## Lifecycle & Ownership

-   **Creation:** This class is never instantiated. As a utility class composed exclusively of static methods, it has no object lifecycle.
-   **Scope:** The class and its static methods are available globally for the duration of the server's runtime, once loaded by the ClassLoader.
-   **Destruction:** Not applicable. The class is unloaded only when the Java Virtual Machine shuts down.

## Internal State & Concurrency

-   **State:** **Stateless and Immutable.** This class holds no member fields and maintains no state between calls. Its behavior is purely a function of its input arguments.

-   **Thread Safety:** The class itself is inherently thread-safe. However, the inventory operations it performs are fundamentally stateful and **not** thread-safe on their own.

    **WARNING:** Thread safety is entirely delegated to the calling context. All mutating operations, such as `internal_removeMaterial`, **must** be executed within a synchronized block or a transactional wrapper provided by the `ItemContainer` instance being modified. The common pattern observed in the source is wrapping calls inside an `itemContainer.writeAction` lambda, which is responsible for acquiring the necessary locks to prevent race conditions and data corruption.

## API Surface

The public API consists of `test` methods used for simulating removal operations without altering container state. The `protected` `internal_` methods perform the actual state mutations and are intended for use only within the `com.hypixel.hytale.server.core.inventory.container` package.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| testRemoveMaterialFromItems(...) | int | O(N) | Simulates removing a material from the entire container. Returns the quantity that could not be removed. |
| getTestRemoveMaterialFromItems(...) | TestRemoveItemSlotResult | O(N) | Simulates removal and returns a detailed result object with slot-level information. |
| testRemoveMaterialFromSlot(...) | int | O(1) | Simulates removing a material from a single, specific slot. Returns the quantity that could not be removed. |
| internal_removeMaterial(...) | MaterialTransaction | O(N) | **(Protected)** Executes the removal of a material from the container, returning a transaction result. |

*N = Capacity of the ItemContainer*

## Integration Patterns

### Standard Usage

This class should not be invoked directly from gameplay systems. The standard and correct pattern is to interact with the public API of an `ItemContainer`. The `ItemContainer` will then use this utility class internally to execute the logic.

The following example illustrates the conceptual internal usage within the `ItemContainer` class, not how an end-user would call it.

```java
// Inside a method of ItemContainer.java
public MaterialTransaction removeMaterial(MaterialQuantity material, boolean allOrNothing) {
    // The ItemContainer uses its writeAction method to ensure thread safety.
    return this.writeAction(() -> {
        // It then calls the internal utility to perform the dispatched logic.
        return InternalContainerUtilMaterial.internal_removeMaterial(
            this,
            material,
            allOrNothing,
            false,
            false
        );
    });
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Invocation from Game Logic:** Calling any method on `InternalContainerUtilMaterial` from systems like quests, AI behaviors, or player commands is an architectural violation. This bypasses the encapsulation and safety guarantees of the `ItemContainer` API.

-   **Unsynchronized Mutation:** Invoking any `internal_` method without wrapping it in the target `ItemContainer`'s `writeAction` or an equivalent locking mechanism. This will lead to severe concurrency issues, including data corruption and server instability.

    ```java
    // DANGEROUS: Bypasses the container's lock
    ItemContainer container = player.getInventory();
    MaterialQuantity toRemove = ...;

    // This call is not synchronized and can corrupt the container's state.
    InternalContainerUtilMaterial.internal_removeMaterial(container, toRemove, true, false, false);
    ```

## Data Pipeline

The primary data flow for this component involves receiving a high-level request from an `ItemContainer`, dispatching it to a specialized handler, and returning a detailed transaction object.

> Flow:
> `ItemContainer.removeMaterial()` call -> `ItemContainer.writeAction()` (Lock Acquired) -> **`InternalContainerUtilMaterial.internal_removeMaterial()`** -> Introspects `MaterialQuantity` type -> Dispatches to `InternalContainerUtilItemStack` / `Tag` / `Resource` -> Modifies `ItemContainer` backing array -> `MaterialTransaction` object returned -> (Lock Released) -> Result propagated to original caller.

