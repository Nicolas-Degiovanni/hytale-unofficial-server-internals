---
description: Architectural reference for UnaryBiPredicate
---

# UnaryBiPredicate

**Package:** com.hypixel.hytale.function.predicate
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface UnaryBiPredicate<J> extends BiPredicate<J, J> {
```

## Architecture & Concepts
UnaryBiPredicate is a specialized functional interface that serves as a semantic and type-safe extension of the standard Java BiPredicate. Its core architectural purpose is to enforce a contract where a predicate must operate on two arguments of the *exact same type*. This eliminates a class of potential runtime ClassCastExceptions and improves code clarity by explicitly signaling the intended use case.

Within the Hytale engine, this interface is a foundational building block for any system requiring the comparison or relational evaluation of two homogeneous objects. It is frequently employed in:
- **Entity Systems:** Comparing two entities (e.g., are they in the same faction?).
- **Inventory Logic:** Evaluating if two ItemStacks can be merged.
- **AI Behavior Trees:** As a condition node checking the relationship between an AI agent and its target.

By extending BiPredicate, it inherits all default methods such as *and*, *or*, and *negate*, allowing for the composition of complex logical conditions from simpler, reusable predicates.

## Lifecycle & Ownership
As a functional interface, UnaryBiPredicate itself does not have a lifecycle. The lifecycle pertains to its *implementations*, which are typically lambda expressions or method references.

- **Creation:** Instances are created ad-hoc via lambda expressions or method references at the point of use. They are not managed by a dependency injection container or factory.
- **Scope:** The lifetime of a UnaryBiPredicate instance is tied to the scope in which it is defined. It may be ephemeral (used and discarded within a single method call) or long-lived (stored as a field in a service or component).
- **Destruction:** Instances are subject to standard Java Garbage Collection. They are eligible for cleanup once they are no longer referenced.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, an implementation (a lambda) can become stateful if it forms a closure by capturing variables from its enclosing scope. Stateless implementations are highly encouraged as they are predictable and reusable.

- **Thread Safety:** The interface contract is inherently thread-safe. The thread safety of a specific *implementation* is entirely dependent on its internal logic and captured state.
    - **Stateless Lambdas:** Implementations that do not capture mutable state are intrinsically thread-safe.
    - **Stateful Lambdas (Closures):** If an implementation captures and modifies non-local variables, it is **not** thread-safe unless access to the captured state is explicitly synchronized. Sharing stateful predicates across threads is a significant concurrency hazard.

    **WARNING:** Extreme caution must be exercised when using stateful UnaryBiPredicate implementations in multi-threaded contexts, such as the parallel entity update loop. Prefer passing state via method arguments rather than capturing it in a closure.

## API Surface
The primary contract is inherited from `java.util.function.BiPredicate<T, U>`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(J t, J u) | boolean | O(N) | Evaluates this predicate on the given arguments. Complexity is dependent on the specific lambda implementation. |
| and(other) | UnaryBiPredicate | O(1) | Returns a composed predicate that represents a short-circuiting logical AND. |
| or(other) | UnaryBiPredicate | O(1) | Returns a composed predicate that represents a short-circuiting logical OR. |
| negate() | UnaryBiPredicate | O(1) | Returns a predicate that represents the logical negation of this predicate. |

## Integration Patterns

### Standard Usage
The interface is intended to be used with lambda expressions to define inline, type-safe comparison logic.

```java
// Example: A predicate to check if two entities are on the same team.
UnaryBiPredicate<Entity> areAllies = (entityA, entityB) -> {
    return entityA.getTeamId() == entityB.getTeamId();
};

Entity player = world.getPlayer();
Entity npc = world.getNpcById(42);

if (areAllies.test(player, npc)) {
    // Do not engage in combat
}
```

### Anti-Patterns (Do NOT do this)
- **Using Raw BiPredicate:** Avoid using a raw `BiPredicate<Object, Object>` and performing manual type casting. This defeats the entire purpose of UnaryBiPredicate and re-introduces the risk of runtime errors.

    ```java
    // BAD: Unsafe and verbose
    BiPredicate<Object, Object> unsafePredicate = (a, b) -> {
        if (a instanceof Entity && b instanceof Entity) {
            return ((Entity) a).getTeamId() == ((Entity) b).getTeamId();
        }
        return false;
    };
    ```

- **Sharing Stateful Implementations:** Creating a predicate that relies on mutable captured state and sharing it across different systems or threads can lead to unpredictable behavior and race conditions.

    ```java
    // BAD: This predicate is stateful and not thread-safe
    int friendlyTeamId = 1;
    UnaryBiPredicate<Entity> isFriendly = (a, b) -> a.getTeamId() == friendlyTeamId && b.getTeamId() == friendlyTeamId;
    // If another thread changes friendlyTeamId, the behavior of isFriendly changes globally.
    ```

## Data Pipeline
UnaryBiPredicate is not a data processing component but a **logical gate**. It consumes two inputs of the same type and produces a single boolean output, typically to control a subsequent logical branch or filter a collection.

> Flow:
> (Input: J, Input: J) -> **UnaryBiPredicate.test()** -> boolean result -> Control Flow (if/else, filter, etc.)

