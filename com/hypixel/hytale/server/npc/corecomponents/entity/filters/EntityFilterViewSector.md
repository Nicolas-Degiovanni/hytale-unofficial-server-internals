---
description: Architectural reference for EntityFilterViewSector
---

# EntityFilterViewSector

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters
**Type:** Transient

## Definition
```java
// Signature
public class EntityFilterViewSector extends EntityFilterBase {
```

## Architecture & Concepts
The EntityFilterViewSector is a server-side predicate component within the Non-Player Character (NPC) Artificial Intelligence framework. Its primary function is to determine if a target entity lies within the frontal field of view of a source NPC. This is a fundamental building block for AI behaviors such as target acquisition, environmental awareness, and stealth detection.

This class operates as a specific filter within a larger entity query or targeting system. It is not a standalone service but rather a data-driven rule that is composed with other filters (e.g., distance checks, faction alignment) to form complex targeting logic.

Architecturally, it integrates directly with the server's Entity Component System (ECS) by querying for `TransformComponent` and `HeadRotation` components. The filter's logic is purely computational, delegating the underlying geometric calculation to the `NPCPhysicsMath` utility. The presence of a `cost` method indicates its participation in a performance-aware query system, allowing the AI engine to optimize complex queries by executing cheaper filters first.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly via the `new` keyword. They are instantiated by the NPC asset loading pipeline, which uses a corresponding `BuilderEntityFilterViewSector` to construct the object from configuration files (e.g., JSON or HOCON definitions of an NPC's behavior).
- **Scope:** The lifetime of an EntityFilterViewSector instance is bound to the specific AI behavior or role that defines it. It persists as long as the parent NPC behavior is active and is discarded if the NPC's configuration is reloaded or the NPC is despawned.
- **Destruction:** Cleanup is managed by the Java Garbage Collector. There are no native resources or explicit destruction methods. The object is reclaimed once it is no longer referenced by any active AI behavior.

## Internal State & Concurrency
- **State:** This object is effectively immutable. Its core state, the `viewCone` angle, is a final field set only during construction. The `matchesEntity` method is stateless with respect to the object itself; it does not modify any internal fields during its execution.
- **Thread Safety:** The class is inherently thread-safe due to its immutable state. However, its methods operate on a shared, mutable `Store<EntityStore>`. Callers are responsible for ensuring that any invocation of `matchesEntity` occurs within a context that guarantees safe, synchronized access to the ECS data, typically the main server thread during an engine tick. The use of `assert` for component presence strongly implies an expectation of single-threaded execution.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesEntity(ref, targetRef, role, store) | boolean | O(1) | Evaluates if the target entity is within the source entity's view cone. Asserts that required components exist. |
| cost() | int | O(1) | Returns the static computational cost of this filter for query optimization purposes. |

## Integration Patterns

### Standard Usage
This filter is not intended to be invoked directly in gameplay code. It is configured as part of an NPC's definition and executed by the AI's targeting or perception system.

```java
// Conceptual example of how the engine might use this filter
// Note: This is a simplified representation. Developers do not write this code.

NPCTargetingSystem targetingSystem = ...;
Ref<EntityStore> npcRef = ...;
List<Ref<EntityStore>> potentialTargets = ...;

// The targeting system would have a list of filters loaded from assets
List<EntityFilterBase> filters = npc.getBehavior().getTargetingFilters();

// The system iterates and applies the filters
for (Ref<EntityStore> target : potentialTargets) {
    boolean isValid = true;
    for (EntityFilterBase filter : filters) {
        // The engine invokes matchesEntity internally
        if (!filter.matchesEntity(npcRef, target, npc.getRole(), entityStore)) {
            isValid = false;
            break;
        }
    }
    if (isValid) {
        // Target is valid
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new EntityFilterViewSector()`. The filter's parameters, like `viewCone`, are derived from game assets. Bypassing the builder-based asset pipeline will lead to unconfigured and non-functional components.
- **Ignoring Component Dependencies:** This filter will throw an `AssertionError` if the source or target entities are missing a `TransformComponent`, or if the source is missing a `HeadRotation` component. Ensure any entity subject to this filter has the required components attached.
- **Misinterpreting 2D Logic:** The underlying calculation in `NPCPhysicsMath.inViewSector` is two-dimensional (X, Z plane). This filter does not account for verticality (Y-axis). Do not rely on it for detecting targets that are directly above or below the NPC.

## Data Pipeline
The filter acts as a processing node in an entity query pipeline. It transforms entity state data into a boolean decision.

> Flow:
> AI Target Selector -> **EntityFilterViewSector.matchesEntity** -> ECS Store Query (for Transform, HeadRotation) -> NPCPhysicsMath.inViewSector (Geometric Test) -> Boolean Result

