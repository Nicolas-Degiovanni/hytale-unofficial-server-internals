---
description: Architectural reference for QuadBoolFunction
---

# QuadBoolFunction

**Package:** com.hypixel.hytale.function.function
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface QuadBoolFunction<T, U, V, W, R> {
```

## Architecture & Concepts
QuadBoolFunction is a specialized functional interface contract. It serves a similar purpose to the standard Java `java.util.function` interfaces but is tailored for a specific arity and type signature: four generic object inputs, one primitive boolean input, and one generic object output.

Its primary architectural role is to provide a standardized, self-documenting type for lambda expressions and method references that fit this exact signature. This avoids the use of more generic but less descriptive types like `Function<Tuple<T, U, V, W, Boolean>, R>`, thereby improving code clarity and avoiding the performance overhead associated with object wrappers for the primitive boolean argument (autoboxing).

This interface is typically used in systems that require highly parameterized callback or strategy patterns, such as in event processing, data transformation pipelines, or complex predicate evaluation where a final boolean flag is a common parameter.

## Lifecycle & Ownership
As an interface, QuadBoolFunction itself does not have a lifecycle. The lifecycle described here pertains to the *implementing instances*, which are almost always lambda expressions or method references.

- **Creation:** An instance is created at the point where a lambda expression or method reference conforming to the interface's signature is declared. It is not managed by any dependency injection container or service registry.
- **Scope:** The lifetime of a QuadBoolFunction instance is tied to the object that holds a reference to it. For example, if a lambda is assigned to a field of a class, it will live as long as the class instance does. If passed as a method argument, its scope may be limited to the duration of that method call.
- **Destruction:** Instances are eligible for garbage collection when they are no longer referenced. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The QuadBoolFunction interface is stateless. However, an implementing lambda expression can be stateful if it is a closure—that is, if it captures variables from its enclosing scope. The state of the captured variables determines the state of the lambda instance.
- **Thread Safety:** The interface itself imposes no concurrency constraints. The thread safety of a specific QuadBoolFunction instance is entirely dependent on its implementation.
    - If the implementation is a pure function (stateless and without side effects), it is inherently thread-safe.
    - If the implementation is a closure that captures and modifies shared, mutable state without proper synchronization, it is **not thread-safe**. Callers must ensure that access to such stateful lambdas is synchronized.

## API Surface
The contract consists of a single abstract method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(T, U, V, W, boolean) | R | O(N) | Executes the function logic. Complexity is entirely dependent on the implementation provided by the developer. |

## Integration Patterns

### Standard Usage
The interface is intended to be implemented using a lambda expression for concise, inline logic definition.

```java
// Example: A function that determines a game event outcome
// based on four context objects and a boolean flag.
QuadBoolFunction<Player, World, Item, Position, EventResult> rule =
    (player, world, item, pos, isCritical) -> {
        if (isCritical && item.isLegendary()) {
            return EventResult.SUCCESS_CRITICAL;
        }
        // ... more logic
        return EventResult.SUCCESS_NORMAL;
    };

// The function is then invoked within a system
EventResult result = rule.apply(p, w, i, pos, true);
```

### Anti-Patterns (Do NOT do this)
- **Anonymous Inner Classes:** While technically possible, using a full anonymous inner class to implement this interface is verbose and defeats the purpose of a lightweight functional interface. Prefer lambdas.
- **Stateful Implementations without Synchronization:** Creating a lambda that modifies a shared, non-thread-safe object from its enclosing scope is a major concurrency risk. Such implementations must not be shared across threads without external locking.
- **Overuse:** Do not use this interface if a standard Java functional interface (e.g., `Function`, `BiFunction`) would suffice. It should be reserved for cases that specifically match its five-argument signature to maintain a clean dependency graph.

## Data Pipeline
This interface does not represent a data pipeline itself, but rather a single, stateless processing node or transformation step *within* a larger pipeline. It is a functional building block.

> Flow:
> Input Data (T, U, V, W, boolean) -> **QuadBoolFunction.apply()** -> Transformed Output (R)

