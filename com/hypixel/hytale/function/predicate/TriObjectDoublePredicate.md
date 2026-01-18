---
description: Architectural reference for TriObjectDoublePredicate
---

# TriObjectDoublePredicate

**Package:** com.hypixel.hytale.function.predicate
**Type:** Behavioral Contract / Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface TriObjectDoublePredicate<T, U, V> {
```

## Architecture & Concepts
TriObjectDoublePredicate is a specialized functional interface designed to encapsulate a boolean-returning condition. It serves as a behavioral contract for logic that evaluates three generic objects and one double-precision floating-point number.

Architecturally, this interface is a key enabler of a functional programming paradigm within the engine. It allows systems to treat conditional logic as data, passing behavior as a parameter to methods. This pattern is fundamental for creating highly decoupled and reusable filtering, querying, and validation systems. It eliminates the need for boilerplate anonymous inner classes for simple, one-off conditional checks.

Common use cases include:
- **Entity Filtering:** Evaluating if an entity meets multiple criteria relative to a player, a world, and a distance.
- **Physics Queries:** Testing if an object's interaction with two other objects at a specific impact force is valid.
- **Procedural Generation:** Applying conditional rules based on a biome type, a chunk coordinate, a neighbor, and a noise value.

## Lifecycle & Ownership
As an interface, TriObjectDoublePredicate itself has no lifecycle. The following pertains to its *implementations*, which are typically lambda expressions or method references.

- **Creation:** Implementations are almost always created ad-hoc at the call site where a method requires this interface. They are defined inline as lambda expressions or passed as method references. Direct class-based implementation is rare and reserved for highly complex, reusable predicates.
- **Scope:** The lifetime of an implementation is typically ephemeral and bound to the execution of the method to which it is passed. If the implementation is a lambda that captures variables (forming a closure), its lifetime may be extended if it is stored by another object.
- **Destruction:** Managed entirely by the Java Garbage Collector. Once the implementation is no longer referenced, it is eligible for collection. Stateless lambdas have a negligible memory footprint.

## Internal State & Concurrency
- **State:** The interface is stateless. Implementations should be designed to be stateless whenever possible. A lambda can become stateful if it captures and/or modifies variables from its enclosing scope. **WARNING:** Stateful predicates introduce side effects and are a significant source of bugs, making system behavior difficult to reason about.
- **Thread Safety:** The contract does not enforce thread safety. The safety of any given implementation is the responsibility of the developer.
    - **Stateless implementations** are inherently thread-safe and can be safely used in parallel streams or multi-threaded job systems.
    - **Stateful implementations** that capture mutable state are **not thread-safe** and must not be shared across threads without explicit synchronization, which defeats the performance benefits of this pattern.

## API Surface
The public contract consists of a single abstract method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(T, U, V, double) | boolean | Implementation-Defined | Executes the predicate's logic. Returns true if the condition is satisfied, false otherwise. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to provide a lambda expression to a higher-order function, such as a filter or a query method.

```java
// Example: Finding nearby entities of a certain type that are hostile to a player.
List<Entity> entities = world.getEntities();
Player player = ...;
double maxDistance = 32.0;

// The lambda is the implementation of TriObjectDoublePredicate<Entity, Player, World>
List<Entity> targets = entities.stream()
    .filter(e -> isViableTarget(e, player, world, maxDistance))
    .collect(Collectors.toList());

// The predicate logic is encapsulated in a method and passed as a reference.
private boolean isViableTarget(Entity entity, Player player, World world, double distance) {
    // Implementation of the TriObjectDoublePredicate
    return entity.isHostileTo(player) && world.hasLineOfSight(entity, player) && distance < 32.0;
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Lambdas:** Avoid implementations that modify external state. This creates side effects and breaks the functional contract of a predicate. The function should be pure, meaning its output depends only on its inputs.
    ```java
    // BAD: This lambda modifies an external counter, creating a side effect.
    int count = 0;
    Predicate<Entity> p = entity -> {
        count++; // Modifying captured state!
        return entity.isAlive();
    };
    ```
- **Overly Complex Logic:** Do not embed deeply nested or multi-stage business logic within a lambda. If a predicate requires more than a few lines of code, encapsulate it in a well-named private method and use a method reference. This maintains readability and testability.

## Data Pipeline
This interface does not process a pipeline of data itself; rather, it acts as a logical gate *within* a data pipeline. It is a component of a larger filtering or processing stage.

> Flow:
> Data Source (e.g., Entity Collection) -> Stream Operation (e.g., filter) -> **TriObjectDoublePredicate Implementation (as gate logic)** -> Filtered Data Set -> Downstream Consumer

