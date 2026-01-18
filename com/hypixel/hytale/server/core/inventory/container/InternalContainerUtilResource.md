---
description: Architectural reference for InternalContainerUtilResource
---

# InternalContainerUtilResource

**Package:** com.hypixel.hytale.server.core.inventory.container
**Type:** Utility

## Definition
```java
// Signature
public class InternalContainerUtilResource {
```

## Architecture & Concepts

The InternalContainerUtilResource class is a stateless, internal utility that provides the core logic for manipulating abstract **Resources** within an ItemContainer. It serves as a low-level implementation detail for the server's inventory system, bridging the conceptual gap between a generic resource request (e.g., "remove 10 wood") and the concrete operations on ItemStacks (e.g., "reduce the count of 3 Oak Log stacks").

This class is not intended for direct use by game logic systems. Instead, it is exclusively invoked by the ItemContainer class to handle the complex calculations and state changes associated with resource-based inventory transactions. Its primary responsibilities include:

1.  **Resource-to-Item Translation:** Determining how many items of a specific type are required to satisfy a given resource quantity, based on the ItemResourceType definitions.
2.  **Transactional State Modification:** Performing removal operations and wrapping the results in detailed transaction objects like ResourceTransaction and ResourceSlotTransaction. This ensures that callers have a complete record of what was changed, what succeeded, and what failed.
3.  **Pre-computation and Validation:** Offering public *test* methods (e.g., testRemoveResourceFromItems) that allow a system to simulate a resource removal. This is critical for implementing **all-or-nothing** logic, where an operation must be guaranteed to succeed fully before any state is actually modified.

The design delegates all concurrency control to the calling ItemContainer, which is expected to wrap invocations within a `writeAction` lambda. This ensures that all modifications are atomic and thread-safe from the perspective of the container.

### Lifecycle & Ownership
- **Creation:** As a static utility class, InternalContainerUtilResource is never instantiated. The Java ClassLoader loads its static members and methods into memory when the class is first referenced.
- **Scope:** The class and its static methods are available for the entire lifetime of the server application.
- **Destruction:** The class is unloaded from memory when the application's ClassLoader is garbage collected, typically during server shutdown.

## Internal State & Concurrency
- **State:** This class is completely **stateless**. It contains no member fields and all its methods are static. The outcome of any method call depends solely on the arguments provided, primarily the state of the passed-in ItemContainer.

- **Thread Safety:** The class itself is inherently thread-safe. However, the operations it performs are fundamentally stateful and **not thread-safe** on their own, as they directly mutate the supplied ItemContainer.

    **WARNING:** Concurrency control is entirely delegated to the caller. All mutating methods (those prefixed with *internal_*) **must** be executed within a synchronized block or lock provided by the ItemContainer instance they are modifying, typically via the `itemContainer.writeAction` pattern. Failure to do so will result in severe data corruption and race conditions.

## API Surface

The API is divided into internal mutator methods and public test methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| internal_removeResourceFromSlot(...) | ResourceSlotTransaction | O(1) | The most granular operation. Attempts to remove a quantity of a resource from a single inventory slot. |
| internal_removeResource(...) | ResourceTransaction | O(N) | Attempts to remove a resource quantity by iterating over all N slots in the container. Aggregates results into a single transaction. |
| internal_removeResources(...) | ListTransaction | O(M\*N) | Processes a list of M resource removal requests against a container with N slots. |
| testRemoveResourceFromItems(...) | int | O(N) | **Simulates** a resource removal without modifying the container. Returns the quantity of the resource that could not be "removed". A return value of 0 indicates the full amount can be satisfied. |

## Integration Patterns

### Standard Usage

This class should only be used internally by the ItemContainer class or closely related inventory systems. The correct pattern involves wrapping the call in the container's write-lock mechanism to ensure atomicity and thread safety.

```java
// Hypothetical example from within ItemContainer.java

public ResourceTransaction removeResource(ResourceQuantity resource, boolean allOrNothing) {
    // The writeAction method acquires a lock, ensuring this entire
    // block is atomic and safe from concurrent modification.
    return this.writeAction(() -> {
        // Delegate the complex logic to the utility class
        return InternalContainerUtilResource.internal_removeResource(
            this,
            resource,
            allOrNothing,
            false,
            true
        );
    });
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation from Game Logic:** Do not call any `internal_` methods from high-level systems like crafting or player commands. These are implementation details. Always use the public API provided on the ItemContainer, which ensures proper locking and validation. Bypassing the container can lead to an inconsistent state.
- **Ignoring Transactional Results:** The methods do not return a simple boolean. They return a rich transaction object. You must inspect this object to understand the outcome. A transaction can succeed but still have a remainder, indicating a partial removal.
    ```java
    // BAD: Assuming the operation worked completely
    ResourceTransaction tx = container.removeResource(myResource, false);
    if (tx.succeeded()) {
        // This is unsafe. The transaction may have only partially succeeded.
        // The full amount might not have been consumed.
    }

    // GOOD: Checking the actual amount consumed
    ResourceTransaction tx = container.removeResource(myResource, false);
    if (tx.getConsumed() > 0) {
        // Logic that depends on the actual amount removed
        log.info("Consumed " + tx.getConsumed() + " of resource.");
    }
    ```
- **Calling Mutators Outside a Lock:** Invoking any `internal_` method without the protection of the container's `writeAction` is a critical error that will break thread safety.

## Data Pipeline

The flow for a typical resource removal operation demonstrates how this utility fits into the broader inventory system.

> Flow:
> High-Level System (e.g., Crafting Recipe) -> `ItemContainer.removeResource(resource)` -> `ItemContainer.writeAction()` [LOCK ACQUIRED] -> **`InternalContainerUtilResource.internal_removeResource()`** -> Modifies `ItemContainer` internal array -> Returns `ResourceTransaction` -> [LOCK RELEASED] -> `ResourceTransaction` returned to High-Level System -> System updates game state based on transaction details.

