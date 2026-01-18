---
description: Architectural reference for EntityFilterLineOfSight
---

# EntityFilterLineOfSight

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters
**Type:** Transient

## Definition
```java
// Signature
public class EntityFilterLineOfSight extends EntityFilterBase {
```

## Architecture & Concepts
The EntityFilterLineOfSight is a concrete implementation of the Strategy Pattern, conforming to the EntityFilterBase contract. Its sole purpose is to act as a high-cost predicate within the server-side NPC entity targeting and threat-assessment systems.

This component is responsible for determining if an unobstructed line of sight exists between two entities. Architecturally, it is a thin wrapper that delegates the computationally expensive raycasting logic to the PositionCache, which is managed by the source entity's Role. This design decouples the *intent* of the filter (checking for line of sight) from the *implementation* of the world query (the raycast itself). This allows the underlying spatial query system to be optimized or swapped without affecting AI behavior logic.

The static COST field is a critical element for the AI's query planner. The high value of 400 signals to the entity selection system that this is an expensive check. The system will typically execute lower-cost filters first (e.g., distance, faction allegiance) to prune the set of potential targets before invoking this filter, thereby minimizing performance impact.

### Lifecycle & Ownership
- **Creation:** Instances are created and managed by higher-level AI behavior systems, often during the parsing and construction of an NPC's behavior tree or targeting profile from configuration files. It is not a managed service or singleton.
- **Scope:** The lifetime of an EntityFilterLineOfSight instance is tied to its owning AI configuration. As the object is stateless, a single instance can be safely shared and reused by all NPCs that employ the same targeting logic.
- **Destruction:** The object is eligible for garbage collection when the AI behavior profile that references it is unloaded or redefined. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It holds no instance fields and its behavior is purely a function of the arguments passed to its methods.
- **Thread Safety:** This class is inherently **thread-safe**. Its stateless nature ensures that it can be called concurrently from multiple threads without risk of race conditions or data corruption. The responsibility for ensuring the thread-safety of the arguments (Ref, Role, Store) lies with the calling system, typically the server's main entity update loop or a dedicated AI job system.

## API Surface
The public API is minimal and focused on fulfilling the EntityFilterBase contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesEntity(ref, targetRef, role, store) | boolean | Complex | Executes the line-of-sight check by delegating to the Role's PositionCache. The complexity is high and dependent on the world's spatial partitioning algorithm. |
| cost() | int | O(1) | Returns the static performance cost of this filter, used for query optimization. |

## Integration Patterns

### Standard Usage
This filter is not intended to be used in isolation. It should be composed into a chain or list of filters within a broader targeting system, which evaluates them in order of cost.

```java
// Hypothetical example of an AI targeting system using the filter
List<EntityFilterBase> targetFilters = new ArrayList<>();
targetFilters.add(new EntityFilterDistance(32.0)); // A cheaper filter
targetFilters.add(new EntityFilterLineOfSight()); // The expensive filter

// The system iterates through potential targets
for (Ref<EntityStore> candidate : potentialTargets) {
    boolean isValid = true;
    for (EntityFilterBase filter : targetFilters) {
        if (!filter.matchesEntity(selfRef, candidate, selfRole, entityStore)) {
            isValid = false;
            break; // Prune target early
        }
    }
    if (isValid) {
        // Add to valid targets list
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Premature Execution:** Do not place this filter at the beginning of a filter chain. Its high cost means it should only run after cheaper checks have already disqualified a significant number of candidates.
- **Direct Invocation in Hot Loops:** Avoid calling matchesEntity directly within performance-critical game loops for broad-phase collision or detection. It is designed for the final stages of targeted AI decision-making, not for general-purpose world queries.

## Data Pipeline
As a predicate, this component acts as a gate in a data flow, not a transformer. It either permits an entity to pass a check or rejects it.

> Flow:
> AI Target Acquisition System -> Provides Candidate Entity -> **EntityFilterLineOfSight** -> [Returns true] -> Candidate is considered visible
>
> AI Target Acquisition System -> Provides Candidate Entity -> **EntityFilterLineOfSight** -> [Returns false] -> Candidate is rejected as obstructed

