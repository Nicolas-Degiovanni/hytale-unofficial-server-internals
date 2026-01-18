---
description: Architectural reference for EntityFilterMovementState
---

# EntityFilterMovementState

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters
**Type:** Transient

## Definition
```java
// Signature
public class EntityFilterMovementState extends EntityFilterBase {
```

## Architecture & Concepts
The EntityFilterMovementState is a highly specialized predicate component within the server-side NPC AI framework. Its sole function is to determine if a target entity currently exhibits a specific MovementState, such as *IDLE*, *WALKING*, or *SPRINTING*.

This class acts as a leaf node in a larger entity querying or targeting system. Multiple filters are often combined to form complex selection criteria for NPC behaviors. For example, an NPC might use this filter to find all other entities that are currently *FLEEING*.

Architecturally, it serves as a bridge between the abstract behavior logic (what an NPC *wants* to check) and the concrete state of the physics and motion simulation (what an entity is *actually* doing). The `cost()` method, which returns zero, is a critical piece of metadata for a higher-level query optimizer. It signals that this check is computationally trivial, likely involving a direct component data lookup and enum comparison, allowing the query planner to execute it early in a filter chain without performance penalty.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly. They are instantiated exclusively by the BuilderEntityFilterMovementState, which is typically configured and invoked by a higher-level system that parses NPC behavior definitions from game data files.
- **Scope:** Extremely short-lived. An EntityFilterMovementState object exists only for the duration of a single query or behavior tree evaluation. It is created on-demand, used to test one or more entities, and then immediately becomes eligible for garbage collection.
- **Destruction:** Managed automatically by the Java Garbage Collector. There are no native resources or explicit cleanup steps required.

## Internal State & Concurrency
- **State:** Immutable. The core state, `movementState`, is a final field set during construction. Once created, an instance of this filter cannot be changed.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. The `matchesEntity` method is a pure function concerning its own state. **Warning:** While the filter object itself is safe, the `Store<EntityStore>` it operates on may not be. Concurrency guarantees for the underlying entity data must be handled by the calling system.

## API Surface
The public contract is minimal, focusing entirely on the filtering operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesEntity(ref, targetRef, role, store) | boolean | O(1) | The primary evaluation method. Returns true if the entity referenced by targetRef is in the configured MovementState. |
| cost() | int | O(1) | Returns the computational cost of this filter for query optimization purposes. Always returns 0. |

## Integration Patterns

### Standard Usage
This filter is not intended to be used in isolation. It is designed to be composed by a larger AI or targeting system, which builds and executes the query.

```java
// Hypothetical usage within a targeting system
// 1. The filter is constructed via its builder, likely from data
BuilderEntityFilterMovementState builder = new BuilderEntityFilterMovementState();
builder.setMovementState(MovementState.WALKING);
EntityFilterMovementState walkingFilter = builder.build();

// 2. The filter is used as a predicate in a stream or query
List<Ref<EntityStore>> allNearbyEntities = ...;
Ref<EntityStore> walkingTarget = allNearbyEntities.stream()
    .filter(target -> walkingFilter.matchesEntity(selfRef, target, selfRole, entityStore))
    .findFirst()
    .orElse(null);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new EntityFilterMovementState()`. The constructor is public to support the builder pattern, but all instantiation must go through the `BuilderEntityFilterMovementState` to ensure correctness and adhere to the intended design.
- **State Caching:** Do not cache the result of `matchesEntity`. The movement state of an entity is highly volatile and can change on any given server tick. The filter must be executed against the live entity state for every query.

## Data Pipeline
The logical flow for this component is straightforward, acting as a simple predicate in a larger data processing chain.

> Flow:
> AI Behavior Tree Node -> **BuilderEntityFilterMovementState** -> Creates **EntityFilterMovementState** instance -> Query Engine executes `matchesEntity` -> `MotionController.isInMovementState` -> Reads component data from `EntityStore` -> Returns boolean result -> AI Behavior Tree acts on result

