---
description: Architectural reference for EntityFilterItemInHand
---

# EntityFilterItemInHand

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters
**Type:** Transient Component

## Definition
```java
// Signature
public class EntityFilterItemInHand extends EntityFilterBase {
```

## Architecture & Concepts
The EntityFilterItemInHand is a specific predicate component within the server-side NPC AI framework. It extends the abstract EntityFilterBase, providing a concrete condition that can be evaluated by higher-level AI systems like Behavior Trees or Goal-Oriented Action Planners.

Its sole responsibility is to determine if a target entity is holding a specific item, or an item from a predefined list, in a designated hand (Main, OffHand, or Both). This allows for the creation of reactive and context-aware NPC behaviors. For example, an NPC guard might become hostile only if a player is holding a weapon, or a friendly NPC might offer a trade if the player is holding a specific quest item.

This class is designed to be data-driven. Its configuration, such as the list of items to check for, is not hard-coded but is supplied by a builder during the NPC asset loading process. The static COST field suggests its use in a performance-aware AI planner that weighs the computational cost of different conditional checks.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly via the new keyword. They are instantiated by the NPC asset pipeline, specifically through a corresponding builder class, BuilderEntityFilterItemInHand. This occurs when an NPC's behavior configuration is loaded from disk.
- **Scope:** The object's lifetime is bound to the lifecycle of the parent AI component that contains it, such as a Role or a specific node in a behavior tree. It is effectively a static data object once created.
- **Destruction:** The object is eligible for garbage collection when the server unloads or reloads the NPC asset configuration it belongs to. There is no manual destruction or cleanup method.

## Internal State & Concurrency
- **State:** The internal state, consisting of the item list and the target hand, is **immutable**. It is set once in the constructor and cannot be changed for the lifetime of the object. The class itself is stateless regarding its configuration.
- **Thread Safety:** This class is **conditionally thread-safe**. Its immutable internal state prevents race conditions within the object itself. However, the matchesEntity method reads from the shared game state via the Store parameter. Safe invocation is dependent on the calling context, which is assumed to be a synchronized game tick or an AI thread that has exclusive or read-safe access to the entity data. It performs no writes to the game state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesEntity(ref, targetRef, role, store) | boolean | O(1) | The core evaluation method. Checks the target entity's inventory against the filter's configured items and hand. |
| cost() | int | O(1) | Returns the static computational cost of this filter, used for AI planning and optimization. |

## Integration Patterns

### Standard Usage
This component is not intended for direct invocation by developers. It is configured within an NPC asset file and used implicitly by the AI engine to evaluate conditions. The engine would retrieve this filter from a behavior definition and execute it.

```java
// Hypothetical AI system evaluating a condition
EntityFilterBase filter = npcBehavior.getCurrentCondition(); // This would be an EntityFilterItemInHand instance

// The engine passes the current context to the filter
boolean conditionMet = filter.matchesEntity(npcRef, playerRef, npcRole, worldStore);

if (conditionMet) {
    // Execute associated behavior, e.g., initiate dialogue
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new EntityFilterItemInHand()`. The object must be constructed via the asset loading system and its corresponding builder to ensure it is correctly configured from the game's data files.
- **External State Management:** Do not cache the result of matchesEntity. The target entity's inventory can change on any game tick, and the check must be performed with live data to be valid.

## Data Pipeline
The component functions as a predicate in a larger data and logic flow, transforming game state into a boolean decision for the AI system.

> Flow:
> NPC Asset File (JSON/HOCON) -> Asset Loader -> **BuilderEntityFilterItemInHand** -> **EntityFilterItemInHand Instance** -> AI System Tick -> `matchesEntity` call -> Read EntityStore for Inventory -> Boolean Result -> AI Decision

