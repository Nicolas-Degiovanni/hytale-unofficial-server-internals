---
description: Architectural reference for EntityFilterMany
---

# EntityFilterMany

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters
**Type:** Abstract Composite Component

## Definition
```java
// Signature
public abstract class EntityFilterMany extends EntityFilterBase implements IAnnotatedComponentCollection {
```

## Architecture & Concepts

EntityFilterMany is an abstract base class that implements the **Composite Design Pattern** for the server-side NPC entity filtering system. It is not a concrete filter itself, but rather a container that treats a collection of individual IEntityFilter components as a single, unified filter. Its concrete subclasses, such as EntityFilterAll (logical AND) and EntityFilterAny (logical OR), use this foundation to build complex targeting and decision-making logic for NPCs.

The primary architectural role of this class is to enable the composition of simple, reusable filters into sophisticated, hierarchical query structures. For example, a simple "IsWithinRange" filter and a "HasLineOfSight" filter can be combined within an EntityFilterMany subclass to create a more complex "IsAttackable" condition.

A critical feature is the performance optimization performed during construction. The component automatically reorders the provided child filters via IEntityFilter.prioritiseFilters. This ensures that the most computationally expensive or highest-impact filters are positioned first. This allows concrete subclasses to perform **short-circuit evaluation**, terminating the filter chain as soon as a definitive result is found, thereby minimizing CPU load during AI ticks. The calculated `cost` is a weighted heuristic that reflects this optimized evaluation order, providing a performance hint to higher-level AI systems.

## Lifecycle & Ownership

-   **Creation:** EntityFilterMany is an abstract class and is never instantiated directly. Its concrete subclasses (e.g., EntityFilterAll) are instantiated by the server's configuration loader when an NPC's behavior, role, or ability definitions are deserialized from asset files.
-   **Scope:** The component's lifetime is strictly bound to its parent component, typically an NPC Role or a node within a Behavior Tree. It persists in memory as part of the NPC's static definition for as long as that NPC type is loaded.
-   **Destruction:** The object is eligible for garbage collection when its parent NPC definition is unloaded from the server. The `unloaded` and `removed` lifecycle methods provide hooks for releasing resources, which are delegated downwards to all contained child filters.

## Internal State & Concurrency

-   **State:** The internal state, consisting of the `filters` array and the calculated `cost`, is **immutable**. Both fields are marked `final` and are fully initialized within the constructor. The object itself is stateless after creation, acting as a delegator.
-   **Thread Safety:** This class is inherently thread-safe due to its immutability. However, this guarantee does not extend to the child IEntityFilter instances it contains. It is the responsibility of the individual filter implementations to ensure their own thread safety. In practice, the NPC AI engine typically operates on a single thread per world or region, mitigating most concurrency concerns.

**WARNING:** Do not assume the entire filter chain is thread-safe. If custom filters with mutable state are used, external synchronization will be required if accessed from multiple threads.

## API Surface

The public contract of EntityFilterMany is primarily concerned with lifecycle event delegation and cost reporting.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EntityFilterMany(List) | constructor | O(N log N) | Constructs the composite. Throws IllegalArgumentException for null inputs. Re-sorts filters and calculates a weighted cost. |
| cost() | int | O(1) | Returns the pre-calculated performance cost heuristic for this entire filter chain. |
| registerWithSupport(Role) | void | O(N) | Delegates the `registerWithSupport` lifecycle event to all contained filters. |
| loaded(Role) | void | O(N) | Delegates the `loaded` lifecycle event to all contained filters. |
| spawned(Role) | void | O(N) | Delegates the `spawned` lifecycle event to all contained filters. |
| unloaded(Role) | void | O(N) | Delegates the `unloaded` lifecycle event to all contained filters. |

## Integration Patterns

### Standard Usage

This class is not used directly. Developers define its behavior declaratively in NPC configuration files (e.g., JSON or HOCON). The system then deserializes these definitions into concrete instances like EntityFilterAll.

A conceptual configuration might look like this:
```json
// Example: Defining a filter for an aggressive mob's targeting
"targetingFilter": {
  "type": "EntityFilterAll", // A concrete subclass of EntityFilterMany
  "filters": [
    {
      "type": "EntityFilterIsHostile"
    },
    {
      "type": "EntityFilterWithinRange",
      "range": 32
    },
    {
      "type": "EntityFilterHasLineOfSight"
    }
  ]
}
```
The game engine would then parse this structure, creating an EntityFilterAll instance that contains the three specified child filters.

### Anti-Patterns (Do NOT do this)

-   **Manual Instantiation:** Do not manually construct EntityFilterMany subclasses in procedural code. Filters are part of an NPC's static data definition and should be configured in assets to allow for hot-reloading and designer iteration.
-   **Modification After Construction:** Do not attempt to modify the internal `filters` array via reflection. The class relies on the immutability of this collection for its performance calculations and thread safety.
-   **Ignoring Lifecycle Events:** When implementing a custom IEntityFilter, it is critical to correctly handle lifecycle events like `loaded` and `unloaded` if your filter needs to cache data or subscribe to other systems. Forgetting to do so can lead to memory leaks or stale data.

## Data Pipeline

EntityFilterMany is not part of a streaming data pipeline but rather a predicate in a decision-making pipeline within the NPC AI tick.

> Flow:
> NPC Behavior Tick -> Target Selection Routine -> **EntityFilterMany Subclass** evaluates target -> Child Filter 1 (e.g., IsHostile) -> Child Filter 2 (e.g., IsInRange) -> ... -> Final Boolean Result -> Behavior Tree Transition

