---
description: Architectural reference for EntityFilterInventory
---

# EntityFilterInventory

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters
**Type:** Component / Strategy

## Definition
```java
// Signature
public class EntityFilterInventory extends EntityFilterBase {
```

## Architecture & Concepts
The EntityFilterInventory is a concrete implementation of the **Strategy Pattern**, designed to function as a server-side predicate within the NPC AI and targeting systems. Its sole purpose is to evaluate whether a target entity's inventory meets a set of predefined criteria.

This component is entirely data-driven, configured through NPC asset definitions and instantiated by the asset loading pipeline. It is not a general-purpose utility but a highly specialized filter used by higher-level systems, such as Behavior Trees or Target Selectors, to make tactical decisions. For example, an NPC might use this filter to find other entities that are carrying specific quest items or have enough free space to receive an item.

The inclusion of a `cost()` method is critical. It integrates with a query optimization layer in the AI engine, allowing the system to build an efficient execution plan. Filters with a lower cost are evaluated first, enabling the system to fail-fast and avoid more computationally expensive checks on entities that do not meet basic criteria.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the NPC asset loading system via a corresponding `BuilderEntityFilterInventory`. This process typically occurs once when the server boots or when NPC assets are hot-reloaded. Direct instantiation during gameplay is a design violation.
- **Scope:** The object is effectively a stateless singleton for a given NPC definition. It is created once from configuration and reused for all subsequent evaluations for any NPC of that type. Its lifetime is tied to the loaded NPC asset collection.
- **Destruction:** The object is marked for garbage collection when the server unloads the associated NPC assets, for instance, during a world unload or server shutdown.

## Internal State & Concurrency
- **State:** **Immutable**. All configurable parameters—such as the item list, count ranges, and free slot requirements—are declared as final fields and are set only once within the constructor. The object holds no mutable state related to any specific entity evaluation.
- **Thread Safety:** **Fully Thread-Safe**. Due to its immutable nature, a single EntityFilterInventory instance can be safely shared and executed by multiple AI routines across different threads without any need for external locking or synchronization. The `matchesEntity` method is a pure function with respect to the object's state.

## API Surface
The public contract is minimal and focused on evaluation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesEntity(ref, targetRef, role, store) | boolean | O(N) | Evaluates the target entity's inventory. Returns true if all criteria are met. N is the number of slots in the target's combined inventory. |
| cost() | int | O(1) | Returns the static computational cost of this filter, used by the AI query planner. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation in typical game logic. It is configured declaratively in asset files and executed by the underlying NPC targeting engine. A conceptual example of how the engine uses it is shown below.

```java
// In a hypothetical NPC targeting system...

// The filter is retrieved from the NPC's pre-loaded behavior definition.
EntityFilterBase inventoryCheck = currentNpc.getBehavior().getFilter("find_player_with_wood");

// The engine iterates through potential targets and applies the filter.
for (Entity target : potentialTargets) {
    boolean isMatch = inventoryCheck.matchesEntity(npcRef, target.getRef(), role, worldStore);
    if (isMatch) {
        // Target is valid, proceed with interaction or behavior.
        npc.setTarget(target);
        break;
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new EntityFilterInventory()`. The object must be constructed via its builder during the asset loading phase to ensure its parameters are correctly initialized from the game's configuration files.
- **Stateful Logic:** Do not attempt to subclass or modify this filter to hold temporary state between calls. The system relies on its stateless and immutable properties for safe concurrent execution.
- **Frequent Player Checks:** Using this filter aggressively on player entities with very large inventories can become a performance bottleneck. Its cost is directly proportional to inventory size.

## Data Pipeline
EntityFilterInventory acts as a gate within a larger data flow for NPC target selection. It does not transform data but rather produces a binary decision that dictates the subsequent flow of logic.

> Flow:
> NPC Behavior Tree Tick -> Target Selector System -> **EntityFilterInventory.matchesEntity()** -> [true] -> Target Selected & Behavior Continues
>
> **or**
>
> NPC Behavior Tree Tick -> Target Selector System -> **EntityFilterInventory.matchesEntity()** -> [false] -> Target Rejected & Selector Tries Next Filter/Entity

