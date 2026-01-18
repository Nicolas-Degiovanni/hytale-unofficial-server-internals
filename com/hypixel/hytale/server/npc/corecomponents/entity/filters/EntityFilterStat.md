---
description: Architectural reference for EntityFilterStat
---

# EntityFilterStat

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters
**Type:** Transient

## Definition
```java
// Signature
public class EntityFilterStat extends EntityFilterBase {
```

## Architecture & Concepts
The EntityFilterStat is a server-side component that functions as a conditional predicate within the NPC behavior and targeting system. It is a specific implementation of the **Strategy Pattern**, where EntityFilterBase defines the contract for various entity filtering conditions.

Its primary role is to determine if a target entity's statistical value (e.g., current health, max mana) falls within a specified range, typically expressed as a ratio of another stat. This allows for the data-driven design of complex AI behaviors, such as:
- An NPC healer targeting allies whose health is below 30% of their maximum health.
- A boss monster entering an enraged state when its own health drops below 50%.
- A spell that only affects targets with more than 90% of their mana remaining.

This class operates exclusively on data retrieved from an entity's EntityStatMap component, making it a read-only processor within the server's Entity-Component-System (ECS) architecture. Its creation is driven by the asset pipeline, meaning its parameters are defined in external configuration files (e.g., JSON) rather than being hardcoded.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly. They are instantiated by the server's asset loading system when parsing NPC behavior definitions. A corresponding builder, BuilderEntityFilterStat, is populated from the asset file, which then constructs the EntityFilterStat object. This process is opaque to the gameplay logic developer.
- **Scope:** The object is immutable and its lifetime is bound to the NPC asset definition it belongs to. It persists in memory as long as the asset is loaded and can be safely shared by all NPC instances of that type.
- **Destruction:** The object is marked for garbage collection when the server unloads the associated NPC asset bundle. There are no manual destruction or cleanup methods.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields, including the stat IDs and value ranges, are final and set exclusively during construction. The object acts as a container for its configuration and has no mutable state.
- **Thread Safety:** **Thread-safe**. Due to its immutable nature, an instance of EntityFilterStat can be safely accessed and used by multiple threads simultaneously without locks or synchronization. The core method, matchesEntity, is a pure function with no side effects on the filter object itself.

**WARNING:** While the class itself is thread-safe, the Store object passed into matchesEntity may not be. Callers are responsible for ensuring that access to the entity data store is properly synchronized if necessary.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesEntity(ref, targetRef, role, store) | boolean | O(1) | Executes the stat comparison logic. Returns true if the target's stat ratio is within the configured min/max bounds. Assumes component lookups are constant time. |
| cost() | int | O(1) | Returns the static performance cost (200) of this filter. Used by the AI system to budget decision-making complexity. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by typical game systems. It is designed to be part of a collection of filters managed by a higher-level AI or targeting system. The system iterates through potential entities and applies the filter to narrow down valid targets or check behavior preconditions.

```java
// Conceptual example within a hypothetical TargetingSystem
List<EntityFilterBase> filters = npcBehavior.getTargetingFilters();
Ref<EntityStore> potentialTarget = ...;

boolean isValidTarget = true;
for (EntityFilterBase filter : filters) {
    // The system invokes matchesEntity polymorphically.
    // If one of the filters is an EntityFilterStat, its logic is run here.
    if (!filter.matchesEntity(npcRef, potentialTarget, npcRole, worldStore)) {
        isValidTarget = false;
        break;
    }
}

if (isValidTarget) {
    // Proceed with action
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new EntityFilterStat()`. The constructor requires a builder object that is only correctly populated by the asset pipeline. Attempting to construct this manually will bypass the intended data-driven design and lead to unmanageable code.
- **Stateful Logic:** Do not extend this class to add mutable state. The entire filter chain relies on these components being stateless and immutable for performance and thread safety.

## Data Pipeline
The flow of data from configuration to execution is a one-way pipeline, ensuring predictable and debuggable NPC behavior.

> Flow:
> NPC Definition Asset (JSON) -> Asset Deserializer -> **BuilderEntityFilterStat** -> **EntityFilterStat Instance** -> AI Targeting System -> `matchesEntity` call -> Read from `EntityStatMap` Component -> Boolean Result

