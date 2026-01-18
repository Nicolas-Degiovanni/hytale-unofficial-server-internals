---
description: Architectural reference for QuadObjectDoublePredicate
---

# QuadObjectDoublePredicate

**Package:** com.hypixel.hytale.function.predicate
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface QuadObjectDoublePredicate<T, U, V, R> {
```

## Architecture & Concepts
QuadObjectDoublePredicate is a specialized, performance-oriented functional interface. It serves as a low-level contract for a predicate—a function that returns a boolean result—that accepts four generic object references and a primitive double value.

This interface is a core component of the engine's custom functional programming extensions. Its primary purpose is to provide a standardized, allocation-free mechanism for complex boolean evaluations in performance-critical systems. By accepting a primitive double, it avoids the overhead associated with boxing the value into a Double object, which is a common performance bottleneck in tight game loops, physics calculations, or complex AI decision trees.

It is not a system or a service, but rather a fundamental building block used by higher-level systems to define conditional logic in a flexible, reusable manner.

## Lifecycle & Ownership
As a functional interface, QuadObjectDoublePredicate itself has no lifecycle. It is a stateless contract. The lifecycle and ownership concerns apply to the *implementations* of this interface, which are typically lambda expressions or anonymous inner classes.

- **Creation:** An implementation is created at the point where a lambda is defined and passed to a method. For example, `(t, u, v, r, d) -> t.isReady() && d > 0.5`.
- **Scope:** The scope of the implementation is tied to the object that holds a reference to it. If it is passed as a method argument, it may be garbage collected after the method returns. If it is stored as a field in a long-lived object, it will persist for the lifetime of that object.
- **Destruction:** The implementation is eligible for garbage collection when no more references to it exist.

**WARNING:** Implementations that capture variables from their enclosing scope (i.e., closures) will hold a reference to the enclosing instance, potentially preventing it from being garbage collected and leading to memory leaks.

## Internal State & Concurrency
- **State:** The interface contract is inherently stateless. However, implementations (lambdas) can be stateful if they close over mutable variables from their surrounding scope. This is a dangerous practice and should be avoided in concurrent environments.
- **Thread Safety:** The interface itself is thread-safe. The thread safety of an *implementation* is entirely the responsibility of the developer who writes it. A lambda that only operates on its input arguments is inherently thread-safe. A lambda that modifies captured state from an outer scope is **not thread-safe** and requires explicit synchronization, which defeats the performance goals of this interface.

**WARNING:** Never capture and modify shared mutable state within an implementation of QuadObjectDoublePredicate that may be executed by a parallel or multi-threaded system. This will lead to race conditions and non-deterministic behavior.

## API Surface
The public contract consists of a single abstract method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(T, U, V, R, double) | boolean | O(N) | Executes the predicate logic. The complexity is determined by the implementation (N). |

## Integration Patterns

### Standard Usage
This interface is intended to be used with lambda expressions for concise, inline definitions of conditional logic. It is typically passed to higher-order functions that perform filtering, querying, or conditional execution.

```java
// Example: A system that filters entities based on player, world, event, and a time delta
public List<Entity> findValidTargets(Player p, World w, GameEvent e, List<Entity> candidates) {
    // The predicate uses four context objects and a primitive double (delta time)
    QuadObjectDoublePredicate<Player, World, GameEvent, Entity> filter =
        (player, world, event, entity, deltaTime) -> {
            return entity.isHostile() && world.canSee(player, entity) && deltaTime > 0.016;
        };

    // A hypothetical utility would use this predicate
    return someSystem.filter(candidates, filter, p, w, e, getDeltaTime());
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Defining a lambda that relies on or modifies external mutable state. This breaks functional purity and creates severe risks in a concurrent engine.
- **Unnecessary Class Implementation:** Creating a full, named class that implements QuadObjectDoublePredicate for a simple, single-use case. This is verbose and defeats the purpose of using a functional interface. Lambdas are strongly preferred.

## Data Pipeline
This interface does not process or transform data. It acts as a conditional gate or filter within a larger data flow.

> Flow:
> System State & Input Arguments -> **QuadObjectDoublePredicate.test()** -> Boolean Result (true/false) -> Conditional Logic Branch (e.g., Add to List / Discard)

