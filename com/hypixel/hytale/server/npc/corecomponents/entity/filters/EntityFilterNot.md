---
description: Architectural reference for EntityFilterNot
---

# EntityFilterNot

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters
**Type:** Transient

## Definition
```java
// Signature
public class EntityFilterNot extends EntityFilterBase implements IAnnotatedComponentCollection {
```

## Architecture & Concepts
The EntityFilterNot class is a fundamental logical operator within the NPC AI's perception system. It functions as a decorator or wrapper that inverts the result of another IEntityFilter component. This adheres to the Composite design pattern, allowing for the construction of complex entity selection criteria by combining simpler, reusable filter components.

Its primary role is to provide a logical NOT operation. For example, instead of creating a specific "FilterNonPlayer" class, a developer can compose `EntityFilterNot` with an existing "FilterPlayer" to achieve the same result with greater flexibility. This component is a critical building block for defining sophisticated NPC behaviors, such as targeting logic, threat assessment, and interaction eligibility. It sits within the decision-making layer of an NPC, helping it to parse and understand its environment by including or excluding entities based on inverted criteria.

## Lifecycle & Ownership
- **Creation:** EntityFilterNot is not instantiated by a central service or manager. It is typically created during the deserialization of NPC configuration data, such as a behavior tree or role definition. A data-driven system reads the NPC's logic, encounters a "not" condition, and instantiates this class, providing the nested filter as a constructor argument.
- **Scope:** The lifetime of an EntityFilterNot instance is tightly coupled to its parent component, usually an NPC Role or a specific behavior node. It persists as long as the parent configuration is loaded and active for an NPC.
- **Destruction:** The object is eligible for garbage collection when its parent Role or behavior is unloaded or the owning NPC is removed from the world. It does not manage any unmanaged resources and relies on standard Java garbage collection for cleanup. The `unloaded` and `removed` methods are lifecycle hooks, not explicit destructors.

## Internal State & Concurrency
- **State:** The class holds a single, final reference to the wrapped IEntityFilter. This makes the EntityFilterNot instance itself immutable after construction. It does not cache the results of its `matchesEntity` calls, ensuring that each evaluation is fresh and reflects the current world state.
- **Thread Safety:** This class contains no internal synchronization mechanisms. Its thread safety is entirely dependent on the execution context provided by the NPC update loop. It is designed to be accessed by a single thread per NPC tick. Concurrent invocation of `matchesEntity` from multiple threads is **not safe** and will lead to race conditions if the wrapped filter or the underlying EntityStore are not designed for concurrent access. All interactions should be confined to the main server thread responsible for NPC updates.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesEntity(ref, targetRef, role, store) | boolean | O(C) | Inverts the result of the wrapped filter's `matchesEntity` call. C is the complexity of the wrapped filter. |
| cost() | int | O(1) | Returns the computational cost of the wrapped filter. Used by the query planner to optimize entity searches. |
| registerWithSupport(role) | void | - | Propagates the lifecycle event to the wrapped filter. |
| loaded(role) | void | - | Propagates the lifecycle event to the wrapped filter. |
| spawned(role) | void | - | Propagates the lifecycle event to the wrapped filter. |
| unloaded(role) | void | - | Propagates the lifecycle event to the wrapped filter. |
| removed(role) | void | - | Propagates the lifecycle event to the wrapped filter. |

## Integration Patterns

### Standard Usage
EntityFilterNot is almost always used to wrap another, more specific filter. It is composed as part of a larger logical structure, typically defined in data and instantiated at runtime.

```java
// Example: Create a filter that matches any entity that is NOT a player.
// This would typically be configured in a data file, not constructed in code.

// 1. Define the filter to be negated
IEntityFilter playerFilter = new EntityFilterPlayer();

// 2. Wrap it with EntityFilterNot
IEntityFilter nonPlayerFilter = new EntityFilterNot(playerFilter);

// 3. Use the composite filter in a behavior
// boolean isMatch = nonPlayerFilter.matchesEntity(...);
```

### Anti-Patterns (Do NOT do this)
- **Null Wrapping:** The constructor does not perform a null check. Passing a null `IEntityFilter` will result in a NullPointerException during any subsequent method call. All wrapped filters must be valid instances.
- **Redundant Nesting:** Avoid wrapping an EntityFilterNot within another EntityFilterNot. A structure like `new EntityFilterNot(new EntityFilterNot(someFilter))` is logically equivalent to `someFilter` but adds unnecessary object overhead and method call depth. Such logic should be simplified.
- **Ignoring Lifecycle Propagation:** The class correctly propagates all lifecycle events (loaded, spawned, etc.) to its wrapped component. Custom parent components that use EntityFilterNot must ensure they call these lifecycle methods, or the nested filters will fail to initialize or de-initialize correctly.

## Data Pipeline
EntityFilterNot acts as a transformation node in a data-processing flow. It does not originate or terminate data but rather alters the logical evaluation of an entity query.

> Flow:
> NPC Behavior Asks "Is Target Valid?" -> Query System Iterates Entities -> **EntityFilterNot**.matchesEntity(entity) -> WrappedFilter.matchesEntity(entity) -> Boolean Result Inverted -> Decision Made

