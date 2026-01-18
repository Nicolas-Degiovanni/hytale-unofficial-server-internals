---
description: Architectural reference for IEntityFilter
---

# IEntityFilter

**Package:** com.hypixel.hytale.server.npc.corecomponents
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface IEntityFilter extends RoleStateChange, IAnnotatedComponent {
```

## Architecture & Concepts
The IEntityFilter interface defines a fundamental contract within the server-side NPC AI system. It represents a single, atomic test or condition used to determine if an entity is relevant to an NPC's current behavior. This interface is a core component of the **Strategy Pattern** used for AI target selection, environmental queries, and decision-making.

Conceptually, an NPC's "brain" or Role does not hardcode its logic for identifying targets. Instead, it is configured with a collection of IEntityFilter implementations. These filters are composed to build complex selection criteria, such as "is an enemy", "is within 10 blocks", "is visible", and "is low on health".

The most critical architectural feature is the `cost()` method. This allows the AI engine to build a highly optimized query pipeline. By executing low-cost filters (e.g., checking a faction tag) before high-cost filters (e.g., performing a raycast for line-of-sight), the system can rapidly discard a majority of non-viable entities, dedicating expensive computational resources only to the most promising candidates. This prioritization is essential for server performance when hundreds of NPCs are active.

## Lifecycle & Ownership
- **Creation:** Implementations of IEntityFilter are typically instantiated as stateless singletons or flyweights during server bootstrap. They are defined as part of an NPC's behavioral configuration, often loaded from asset files or registered programmatically by AI behavior modules. They are not created per-NPC or per-frame.
- **Scope:** An IEntityFilter instance is designed to be long-lived, persisting for the entire server session. Its stateless nature allows a single instance to be shared and reused across thousands of NPC agents simultaneously.
- **Destruction:** Instances are garbage collected when the server shuts down or if the AI behavior definitions are dynamically reloaded.

## Internal State & Concurrency
- **State:** The IEntityFilter contract is inherently stateless. Implementations **must not** maintain any mutable state that changes based on the inputs to `matchesEntity`. They should be pure functions whose output depends solely on the arguments provided.
- **Thread Safety:** The contract is designed for a highly concurrent environment. The server's AI engine will invoke `matchesEntity` from multiple worker threads, each processing a different NPC. All implementations of this interface **must be unconditionally thread-safe**. Failure to ensure this will result in severe, difficult-to-diagnose race conditions and AI misbehavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesEntity(Ref, Ref, Role, Store) | boolean | Varies | The core evaluation method. Returns true if the target entity (var2) matches the filter's criteria relative to the source NPC (var1). |
| cost() | int | O(1) | Returns the computational cost of this filter. Used for query optimization. |
| prioritiseFilters(IEntityFilter[]) | static void | O(N log N) | An in-place utility method to sort an array of filters from lowest to highest cost. |

## Integration Patterns

### Standard Usage
The primary integration pattern involves an AI controller or a targeting system composing a chain of filters to find a suitable entity. The engine first sorts the filters by cost, then executes them sequentially until one fails, short-circuiting the evaluation.

```java
// An AI system retrieves a set of filters for its current task
IEntityFilter[] filters = currentRole.getAcquisitionFilters();

// CRITICAL: Always prioritise filters to ensure performance
IEntityFilter.prioritiseFilters(filters);

// Iterate through potential entities provided by a broad-phase query
for (Ref<EntityStore> potentialTarget : world.getNearbyEntities(self)) {
    boolean allMatch = true;
    for (IEntityFilter filter : filters) {
        // If any filter fails, this target is invalid; break early
        if (!filter.matchesEntity(self, potentialTarget, currentRole, entityStore)) {
            allMatch = false;
            break;
        }
    }

    if (allMatch) {
        // This entity is a valid target
        setSelectedTarget(potentialTarget);
        return;
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Creating a filter that caches results or modifies internal fields within the `matchesEntity` method is a critical architectural violation. This breaks reusability and thread safety.
- **Ignoring Cost:** Implementing a filter without providing an accurate, non-zero `cost()` is a performance anti-pattern. An expensive filter with a cost of 0 may be executed before a cheap one, degrading server performance.
- **Manual Sorting:** Avoid writing custom sorting logic. Always use the static `IEntityFilter.prioritiseFilters` helper to ensure the sorting algorithm is consistent with engine expectations.

## Data Pipeline
The IEntityFilter interface sits in the middle of the AI's entity evaluation pipeline. It acts as the primary mechanism for transforming a broad set of potential entities into a narrow, actionable set of candidates.

> Flow:
> AI Behavior Tick -> Broad-Phase World Query (e.g., "entities in radius") -> **IEntityFilter Chain Execution** -> Filtered List of Valid Entities -> Final Target Selection Logic

