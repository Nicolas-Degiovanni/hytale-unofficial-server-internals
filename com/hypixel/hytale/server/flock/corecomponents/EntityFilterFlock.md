---
description: Architectural reference for EntityFilterFlock
---

# EntityFilterFlock

**Package:** com.hypixel.hytale.server.flock.corecomponents
**Type:** Strategy Object / Transient

## Definition
```java
// Signature
public class EntityFilterFlock extends EntityFilterBase {
```

## Architecture & Concepts
The EntityFilterFlock is a concrete implementation of the `EntityFilterBase` contract, designed to function as a highly specific predicate within the server's AI and NPC behavior systems. Its core purpose is to evaluate whether a given entity meets a set of criteria related to its participation in a "flock"—a server-side grouping of entities, often led by a player or another NPC.

This class embodies the **Strategy** design pattern. It encapsulates a complex set of logical conditions—such as flock leadership status, flock size, and the nature of the flock's leader (player or NPC)—into a single, immutable, and reusable object. Higher-level AI systems, such as behavior trees or target acquisition services, compose and execute these filters to make tactical decisions without needing to know the intricate details of flock mechanics.

The filter's `cost` method, which returns a static value, suggests its integration into a query optimization system that may execute cheaper filters before more expensive ones to fail-fast and conserve server resources during complex AI evaluations.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively via the `BuilderEntityFilterFlock`. This builder is typically configured and invoked by a factory or parser that translates game design data (e.g., JSON behavior definitions) into executable AI logic at runtime. Direct instantiation is not the intended use case.
- **Scope:** **Transient and short-lived**. An EntityFilterFlock object is designed to exist only for the duration of a single AI evaluation tick or a specific target query. It holds no persistent state and is not intended to be cached or reused across different evaluation contexts.
- **Destruction:** The object is managed by the Java garbage collector. Once the AI system that created it completes its evaluation and discards its reference, the filter instance becomes eligible for garbage collection. No manual cleanup is necessary.

## Internal State & Concurrency
- **State:** **Immutable**. All configuration fields (`flockMembership`, `size`, `checkCanJoin`, etc.) are declared `final` and are initialized only once during construction via the builder. The object does not modify its internal state after creation, nor does it cache any data from the world state.
- **Thread Safety:** **Conditionally Thread-Safe**. The class itself is inherently thread-safe due to its immutability. An instance can be safely passed between threads. However, the `matchesEntity` method operates on a `Store<EntityStore>` object, which represents the live game world state. The thread safety of an entire filtering operation is therefore dependent on the concurrency guarantees of the `Store` implementation provided to it. The filter itself introduces no race conditions.

## API Surface
The public contract is minimal, focusing entirely on the execution of the filter logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesEntity(ref, targetRef, role, store) | boolean | O(1) | Evaluates the target entity against the filter's configured flocking rules. Relies on several component lookups which are assumed to be constant time. |
| cost() | int | O(1) | Returns the static computational cost of this filter, used for query planning by AI systems. |

## Integration Patterns

### Standard Usage
EntityFilterFlock is not intended to be used in isolation. It is constructed by a builder and passed as a predicate to a higher-level entity querying or behavior-selection system.

```java
// Hypothetical usage within a target selection system

// 1. Configure the filter using its builder
BuilderEntityFilterFlock filterBuilder = new BuilderEntityFilterFlock();
filterBuilder.setFlockMembership(FlockMembershipType.Follower);
filterBuilder.setSize(new int[]{5, 10}); // Target flocks with 5 to 10 members
filterBuilder.setFlockPlayerMembership(FlockPlayerMembership.Member); // Leader must be a player

EntityFilterFlock flockFilter = filterBuilder.build();

// 2. Use the filter in a broader query
// The 'entityQuerySystem' would iterate potential targets and apply the filter.
List<Ref<EntityStore>> validTargets = entityQuerySystem.findEntities(
    originEntity,
    searchRadius,
    flockFilter // The filter is passed as a condition
);
```

### Anti-Patterns (Do NOT do this)
- **Bypassing the Builder:** Do not attempt to construct an EntityFilterFlock directly or modify its fields via reflection. The system relies on the immutability guarantees provided by the builder pattern for safe and predictable behavior.
- **Caching Instances:** Avoid storing and reusing instances of this filter across different AI agents or over long periods. They are lightweight and cheap to create, and their configuration is often context-specific. Caching can lead to subtle bugs where an AI agent uses a stale or incorrect set of filtering criteria.
- **Misinterpreting Thread Safety:** While the object is immutable, calling `matchesEntity` with a non-thread-safe `Store` from multiple threads will lead to severe concurrency issues. The safety of the operation is dictated by the safety of the `Store` argument.

## Data Pipeline
The primary data flow occurs during the `matchesEntity` evaluation. The filter acts as a gate, transforming a target entity reference into a boolean decision.

> Flow:
> AI Behavior Tree Node -> Entity Query System -> **EntityFilterFlock.matchesEntity(target)** -> Component Lookups (FlockMembership, EntityGroup) from ECS Store -> Rule Evaluation -> **boolean result** -> AI Behavior Tree Node

