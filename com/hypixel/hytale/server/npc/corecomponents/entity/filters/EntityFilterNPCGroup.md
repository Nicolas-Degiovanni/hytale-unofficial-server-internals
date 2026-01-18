---
description: Architectural reference for EntityFilterNPCGroup
---

# EntityFilterNPCGroup

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters
**Type:** Transient Strategy Object

## Definition
```java
// Signature
public class EntityFilterNPCGroup extends EntityFilterBase {
```

## Architecture & Concepts
The EntityFilterNPCGroup is a concrete implementation of the *Strategy* design pattern, providing a specific algorithm for filtering game entities based on their NPC group affiliation. This component is a fundamental building block within the server-side AI perception and targeting systems.

Its primary function is to determine if a target entity should be included or excluded from a candidate set based on pre-configured lists of group IDs. It operates exclusively on integer-based group identifiers for high performance during the game loop. The translation from human-readable group names (defined in asset files) to these integer IDs is handled upstream by the asset loading pipeline, specifically by the BuilderEntityFilterNPCGroup.

A key architectural feature is the static `cost` value. The AI scheduler or behavior tree executor can use this cost to optimize query plans. For example, it may choose to run cheaper filters first to reduce the set of entities that must be processed by more computationally expensive filters like this one.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the NPC asset loading system via a corresponding builder, BuilderEntityFilterNPCGroup. This process typically occurs when the server loads or hot-reloads NPC definition files from disk. It is never instantiated directly in game logic.
- **Scope:** The lifetime of an EntityFilterNPCGroup object is bound to its owning component, such as a behavior tree node or a targeting profile. It is a stateless, configuration-driven object that persists as long as its parent asset is loaded in memory.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for collection once its owning asset is unloaded and no longer referenced. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state consists of two final integer arrays: `includeGroups` and `excludeGroups`. These arrays are populated once during construction and cannot be modified thereafter. This immutability is a critical design choice for ensuring predictable behavior and thread safety.
- **Thread Safety:** **Thread-safe**. Due to its immutable nature, an instance of EntityFilterNPCGroup can be safely shared and accessed by multiple threads simultaneously without locks or synchronization. The `matchesEntity` method is a pure function relative to the object's internal state, meaning its output depends only on its inputs.

**WARNING:** While the class itself is thread-safe, the `Store<EntityStore>` object passed into `matchesEntity` must be accessed in a thread-safe manner by the underlying `WorldSupport` utility.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesEntity(ref, targetRef, role, store) | boolean | O(N) | The core filtering logic. Returns true if the target entity satisfies the include/exclude group rules. Complexity is proportional to the number of groups configured in the filter. |
| cost() | int | O(1) | Returns the static computational cost of this filter, used for query optimization by the AI system. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by gameplay programmers. It is configured within an NPC asset file and executed by the underlying AI engine. The following example illustrates how a system, such as a target acquisition module, would use an instance of this filter.

```java
// Hypothetical system-level usage
EntityFilterBase filter = loadedBehavior.getTargetingFilter(); // The filter is retrieved from a loaded asset

// The system iterates through potential targets
for (Ref<EntityStore> potentialTarget : allEntitiesInVicinity) {
    if (filter.matchesEntity(selfRef, potentialTarget, selfRole, entityStore)) {
        // Target is valid, add to candidate list
        validTargets.add(potentialTarget);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new EntityFilterNPCGroup()`. This bypasses the critical asset-building process where group names are resolved into integer IDs. An instance created this way will be non-functional.
- **State Modification:** Do not attempt to modify the internal `includeGroups` or `excludeGroups` arrays via reflection. This violates the immutability contract and can lead to unpredictable behavior across the AI system.

## Data Pipeline
The EntityFilterNPCGroup acts as a boolean predicate within a larger data processing pipeline, typically for target selection.

> Flow:
> AI Behavior Tree Tick -> Target Acquisition System -> **EntityFilterNPCGroup.matchesEntity(candidate)** -> Boolean Result -> Add to Valid Target List

