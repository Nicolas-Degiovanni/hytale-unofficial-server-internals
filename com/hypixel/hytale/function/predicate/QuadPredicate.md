---
description: Architectural reference for QuadPredicate
---

# QuadPredicate

**Package:** com.hypixel.hytale.function.predicate
**Type:** Functional Interface

## Definition
```java
// Signature
public interface QuadPredicate<T, R, S, U> {
```

## Architecture & Concepts
QuadPredicate is a functional interface that serves as a fundamental building block for expressing conditional logic involving four distinct inputs. It is an extension of the standard Java Predicate pattern, generalized to accept four arguments.

Within the Hytale engine, this interface is primarily used to define complex filtering criteria, validation rules, or conditional triggers in systems where multiple entities or data points must be evaluated simultaneously. For example, it could be used to determine if a player (T) can interact with an object (R) at a specific location (S) under certain world conditions (U).

By encapsulating a boolean test as an object, QuadPredicate promotes a declarative, functional programming style. This allows complex logic to be passed as a parameter to higher-order functions, decoupling the "what" (the rule) from the "how" (the execution context).

## Lifecycle & Ownership
As an interface, QuadPredicate itself has no lifecycle. The following applies to concrete *implementations* of the interface, which are typically created as lambda expressions or anonymous inner classes.

- **Creation:** Implementations are created on-demand at the call site. They are often instantiated to be passed as arguments to methods that require a conditional check.
- **Scope:** The lifetime of a QuadPredicate implementation is tied to the object that holds a reference to it. It may be short-lived (scoped to a single method call) or long-lived (stored as a field in a service or component).
- **Destruction:** Instances are managed by the Java Garbage Collector. They are eligible for cleanup once they are no longer reachable, for example, after the method they were passed to completes, or when the containing object is destroyed.

## Internal State & Concurrency
- **State:** The interface contract is stateless. However, implementations can be stateful if they are defined using a lambda that closes over mutable variables from its enclosing scope. This is a powerful but potentially hazardous pattern.

- **Thread Safety:** The interface itself provides no guarantees of thread safety. The responsibility lies entirely with the implementation. A stateless implementation (e.g., a pure function lambda) is inherently thread-safe. A stateful implementation that captures and modifies shared, mutable state is **not** thread-safe and must be synchronized externally if used concurrently.

**WARNING:** Avoid creating stateful QuadPredicate implementations that are shared across multiple threads. This can lead to severe and difficult-to-diagnose race conditions. Prefer pure functions for predicates wherever possible.

## API Surface
The public contract consists of a single abstract method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(T, R, S, U) | boolean | Implementation-Defined | Evaluates this predicate on the given arguments. Returns true if the inputs match the predicate, otherwise false. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to pass a QuadPredicate implementation (usually as a lambda) to a higher-order function that performs filtering or conditional execution.

```java
// Example: A system that filters a list of potential targets
// based on a player, a weapon, a position, and world state.

public List<Target> findValidTargets(Player p, Weapon w, Position pos, WorldState state, QuadPredicate<Player, Weapon, Position, WorldState> filter) {
    List<Target> results = new ArrayList<>();
    for (Target target : allPotentialTargets) {
        // The filter logic is provided by the caller
        if (filter.test(p, w, pos, state)) {
            results.add(target);
        }
    }
    return results;
}

// Caller provides the specific logic
List<Target> validTargets = findValidTargets(player, weapon, position, world,
    (p, w, pos, state) -> p.hasLineOfSight(pos) && w.hasEnoughAmmo() && state.isDayTime()
);
```

### Anti-Patterns (Do NOT do this)
- **Complex Logic:** Do not embed time-consuming operations like I/O, network requests, or heavy computation within the `test` method. Predicates are expected to be fast, lightweight checks.
- **Stateful Lambdas in Concurrent Systems:** Avoid using a predicate that modifies external state, especially if it might be used by multiple threads. This breaks the functional paradigm and introduces side effects.

```java
// ANTI-PATTERN: Modifying external state from a predicate
List<String> errors = new ArrayList<>();
someStream.filter((a, b, c, d) -> {
    if (!isValid(a)) {
        errors.add("Invalid A"); // DANGEROUS SIDE EFFECT
        return false;
    }
    return true;
});
```

## Data Pipeline
QuadPredicate does not manage a data pipeline itself. Instead, it acts as a conditional **gate** or **filter** within a larger data flow. It consumes four data inputs and produces a single boolean output that determines the subsequent path of the data.

> Flow:
> Input Stream (T, R, S, U) -> **QuadPredicate.test()** -> [true] -> Continue Pipeline / [false] -> Discard or Divert


