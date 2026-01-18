---
description: Architectural reference for NoDuplicateFilter
---

# NoDuplicateFilter

**Package:** com.hypixel.hytale.server.core.inventory.container.filter
**Type:** Transient

## Definition
```java
// Signature
public class NoDuplicateFilter implements ItemSlotFilter {
```

## Architecture & Concepts
The NoDuplicateFilter is a concrete implementation of the **Strategy Pattern**, designed to enforce a specific inventory rule: preventing more than one item of a given type from being placed into a container. It acts as a predicate or a gate within the server-side inventory system.

This class is not a standalone service. Instead, it is tightly coupled to a specific instance of a SimpleItemContainer. Its sole responsibility is to inspect the state of its associated container and determine if adding a proposed Item would violate the "no duplicates" constraint. This design decouples the container's core logic of storing items from the specific business rules that govern which items are allowed, enabling flexible and composable inventory behaviors.

## Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level system responsible for configuring an inventory container, such as a ContainerFactory or specific game logic for a special-purpose inventory (e.g., a trophy case). It is always created with a direct reference to the container it will manage.
- **Scope:** The lifecycle of a NoDuplicateFilter instance is strictly bound to the SimpleItemContainer instance it was constructed with. It does not persist beyond the life of its container.
- **Destruction:** The object is managed by the Java Garbage Collector. There are no manual cleanup or disposal methods. It becomes eligible for collection once the container it references is destroyed and no other references to the filter exist.

## Internal State & Concurrency
- **State:** The NoDuplicateFilter is stateful by reference. It holds a final reference to a SimpleItemContainer. While the filter's own fields are immutable, its behavior is entirely dependent on the mutable state of the referenced container.
- **Thread Safety:** **This class is not thread-safe.** The test method iterates over the container's contents without any internal locking. If another thread modifies the container while the test method is executing, the behavior is undefined and may lead to inconsistent results. All interactions with this filter and its associated container must be synchronized externally by the owning inventory system.

## API Surface
The public contract is minimal, consisting only of the interface method it implements.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(Item item) | boolean | O(N) | Scans the entire associated container. Returns true if the item is not a duplicate and can be added; returns false if a duplicate is found or the input item is null. |

## Integration Patterns

### Standard Usage
This filter is intended to be used as a behavioral component for an inventory slot or container. It is created and immediately assigned to the system that will use it to validate item placement operations.

```java
// Correctly create a filter for a specific container
SimpleItemContainer trophyCase = new SimpleItemContainer(5);
ItemSlotFilter filter = new NoDuplicateFilter(trophyCase);

// The inventory system would then use this filter before placing an item
if (filter.test(newItem)) {
    trophyCase.addItem(newItem);
}
```

### Anti-Patterns (Do NOT do this)
- **State Misinterpretation:** Do not assume the filter's result is static. Calling test at time T1 gives no guarantee about the result at time T2 if the underlying container could have been modified in the interim. The check and the subsequent modification must be performed as an atomic operation.
- **External Modification During Test:** Never allow another thread to modify the container while the test method is being executed. This will lead to race conditions. The inventory system must enforce a strict locking policy.

## Data Pipeline
The NoDuplicateFilter acts as a conditional gate in the data flow of an item-placement operation. It does not transform data but rather halts or permits the flow based on a state check.

> Flow:
> Player Action (e.g., Move Item) -> Server Inventory Logic -> **NoDuplicateFilter.test(item)** -> [Conditional: Allow/Deny] -> SimpleItemContainer State Change -> Network Sync to Client

