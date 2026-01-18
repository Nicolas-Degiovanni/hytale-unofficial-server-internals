---
description: Architectural reference for EntityFilterAnd
---

# EntityFilterAnd

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters
**Type:** Transient

## Definition
```java
// Signature
public class EntityFilterAnd extends EntityFilterMany {
```

## Architecture & Concepts
The EntityFilterAnd class is a concrete implementation of the **Composite Design Pattern**, designed to function as a logical **AND** gate for entity selection. It is a fundamental building block within the server's NPC AI and entity querying systems, enabling the creation of complex, multi-faceted filtering rules by combining simpler, single-purpose filters.

Architecturally, it acts as a container node in a filter tree. Instead of implementing a specific filtering logic itself, it delegates the evaluation to a collection of child IEntityFilter objects. Its sole responsibility is to enforce the logical AND condition: an entity is considered a match only if it satisfies the criteria of *every* filter contained within this component.

This approach promotes modularity and reusability. Complex targeting criteria, such as "find a hostile entity that is within a 10-block radius AND has less than 50% health," can be assembled dynamically from individual components like FilterIsHostile, FilterIsInRange, and FilterHasLowHealth without requiring a new, monolithic filter class.

## Lifecycle & Ownership
- **Creation:** EntityFilterAnd instances are not managed by a central service or registry. They are typically instantiated on-demand by higher-level systems, such as a Behavior Tree node, a quest objective script, or an AI role configuration loader. It is constructed with a pre-populated list of IEntityFilter implementations.
- **Scope:** The lifetime of an EntityFilterAnd object is ephemeral and is strictly tied to its owner. For example, if it is part of an NPC's "attack" behavior, it exists only as long as that behavior is active.
- **Destruction:** The object is eligible for garbage collection as soon as its owning component is discarded. It manages no persistent resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The component's state consists of the list of child filters provided during construction. This list is stored in the parent EntityFilterMany class and is not modified after instantiation. Therefore, an EntityFilterAnd instance is effectively immutable.
- **Thread Safety:** This class is inherently thread-safe. The matchesEntity method is a pure function with no side effects; it only reads from its internal, immutable list of filters and the arguments passed to it.

    **WARNING:** While the EntityFilterAnd class itself is thread-safe, the overall thread safety of an evaluation depends on the thread safety of the child IEntityFilter implementations it contains. It is critical that all composed filters are also stateless and thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesEntity(ref, targetRef, role, store) | boolean | O(N) | Evaluates a target entity against all contained filters. Returns true if and only if all child filters return true. Short-circuits and returns false on the first failure. N is the number of contained filters. |

## Integration Patterns

### Standard Usage
EntityFilterAnd is used to combine multiple simple filters into a single, sophisticated rule. It is almost always used as part of a larger collection or configuration that defines an NPC's behavior.

```java
// Example: Define a filter that targets low-health enemies nearby.
List<IEntityFilter> rules = new ArrayList<>();
rules.add(new FilterIsHostile());
rules.add(new FilterIsInRange(15.0));
rules.add(new FilterHasHealthPercent(0.5, Comparison.LESS_THAN));

// The EntityFilterAnd combines these rules.
IEntityFilter compositeFilter = new EntityFilterAnd(rules);

// Later, the AI system uses the composite filter to find a target.
if (compositeFilter.matchesEntity(selfRef, potentialTargetRef, role, store)) {
    // Execute attack logic...
}
```

### Anti-Patterns (Do NOT do this)
- **Empty Filter List:** Instantiating EntityFilterAnd with an empty list of filters is a logical flaw. The matchesEntity method will always return true, effectively disabling any filtering. This usually indicates a configuration error and should be guarded against by the calling system.
- **Nesting Redundant Filters:** Nesting an EntityFilterAnd within another EntityFilterAnd provides no logical benefit and adds unnecessary object overhead and processing depth. Filter lists should be flattened before being passed to the constructor.
- **Using Mutable Filters:** The component assumes its child filters are stateless. Passing a filter that modifies its internal state during an evaluation can lead to non-deterministic behavior, especially in a multi-threaded environment. This pattern is highly unstable and must be avoided.

## Data Pipeline
EntityFilterAnd acts as a predicate or gate within a larger data processing flow, typically for entity selection. It does not transform data but rather determines whether a given data item (an entity) should proceed through the pipeline.

> Flow:
> AI Behavior System -> Query for Potential Targets -> [List of Entities] -> ForEach Entity -> **EntityFilterAnd.matchesEntity()** -> [true/false] -> Add to Final Target List -> Action Execution

