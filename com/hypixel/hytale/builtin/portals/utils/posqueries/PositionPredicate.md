---
description: Architectural reference for PositionPredicate
---

# PositionPredicate

**Package:** com.hypixel.hytale.builtin.portals.utils.posqueries
**Type:** Functional Interface

## Definition
```java
// Signature
public interface PositionPredicate {
```

## Architecture & Concepts
PositionPredicate defines a stateless contract for evaluating a specific condition at a given 3D coordinate within a World. It embodies the Strategy Pattern, abstracting the *logic* of a spatial test from the *execution* of that test.

This interface is a fundamental component of the server's spatial query system. Systems that need to find valid locations for game elements—such as portal generation, structure placement, or AI pathfinding—leverage implementations of PositionPredicate to filter vast sets of candidate positions. By composing multiple predicates, complex placement rules can be built in a modular and reusable fashion. For example, one predicate might check for solid ground, while another checks for sufficient vertical clearance.

Its primary architectural role is to provide a flexible, lambda-friendly mechanism for injecting custom validation logic into core world-querying algorithms.

### Lifecycle & Ownership
- **Creation:** Implementations of this interface are typically created on-the-fly as anonymous classes or, more commonly, as lambdas. They are instantiated by higher-level services that orchestrate a spatial query.
- **Scope:** The lifecycle of a PositionPredicate instance is almost always transient and scoped to a single method call. It exists only for the duration of the query operation it is part of.
- **Destruction:** The instance is eligible for garbage collection as soon as the querying method returns and no other references to it exist.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, implementations may be stateful. For example, a lambda can close over variables from its enclosing scope, effectively giving it state.

- **Thread Safety:** The contract makes no guarantees about thread safety. Concurrency concerns are entirely the responsibility of the implementation.

    **WARNING:** Predicates may be executed in parallel by world generation or query systems. Implementations **must** be thread-safe if they access or modify shared state. Prefer purely functional, stateless implementations to avoid concurrency hazards.

## API Surface
The public contract consists of a single method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(World world, Vector3d pos) | boolean | *Varies* | Evaluates the predicate at the given position in the specified world. Returns true if the condition is met, false otherwise. Complexity is implementation-dependent. |

## Integration Patterns

### Standard Usage
The most common pattern is to define the logic as a lambda and pass it to a utility method that performs a search or validation operation.

```java
// Example: Finding a valid spawn point using a predicate
PositionQueryService queryService = context.getService(PositionQueryService.class);

// Predicate to ensure the block is air and the block below is grass
PositionPredicate canSpawnHere = (world, pos) -> {
    BlockState above = world.getBlockState(pos);
    BlockState below = world.getBlockState(pos.down());
    return above.isAir() && below.isGrass();
};

Optional<Vector3d> spawnPoint = queryService.findFirstValid(area, canSpawnHere);
```

### Anti-Patterns (Do NOT do this)
- **Blocking Operations:** Never perform file I/O, database queries, or network requests within a predicate's test method. These methods are often called in tight loops during world generation or chunk loading and must execute instantaneously.
- **Complex State Management:** Avoid creating predicates that rely on complex, mutable external state. This is a strong indicator of a design flaw and can lead to unpredictable behavior and race conditions, especially if the query system parallelizes its work.
- **Ignoring World Context:** The provided World object is the single source of truth for the query. Do not attempt to access world state through other means, as it may be inconsistent with the snapshot being used for the query.

## Data Pipeline
PositionPredicate acts as a filter or gate within a larger data processing pipeline, typically for spatial queries.

> Flow:
> High-Level System (e.g., StructureGenerator) -> Defines Search Area -> Query Engine -> Iterates Candidate Positions -> **PositionPredicate.test()** -> Filters out invalid positions -> Returns a list of valid Vector3d coordinates

