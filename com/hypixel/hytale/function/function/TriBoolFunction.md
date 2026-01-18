---
description: Architectural reference for TriBoolFunction
---

# TriBoolFunction

**Package:** com.hypixel.hytale.function.function
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface TriBoolFunction<T, U, V, R> {
   R apply(T var1, U var2, V var3, boolean var4);
}
```

## Architecture & Concepts
TriBoolFunction is a specialized, stateless functional interface that defines a contract for a function that accepts three generic object arguments and one primitive boolean, producing a single generic result.

Its primary architectural purpose is to provide a standardized, high-performance mechanism for passing behavior as data, particularly within performance-sensitive systems like game logic, rendering pipelines, or data processing tasks. By including a primitive boolean directly in the signature, it avoids the overhead of boxing the boolean into an object and the allocation of a temporary container object (e.g., a Pair or custom tuple) to group the four arguments.

This interface is a foundational element of a functional programming paradigm within the Hytale engine, encouraging developers to write more declarative, decoupled, and testable code by abstracting logic into reusable function objects.

## Lifecycle & Ownership
As a functional interface, TriBoolFunction itself does not have a lifecycle. Instead, the lifecycle pertains to the *implementations* of the interface, which are typically lambda expressions or method references.

- **Creation:** Implementations are created inline at their point of use. For example, `(t, u, v, b) -> new Result(t, b)` creates a new instance of an anonymous class that implements TriBoolFunction.
- **Scope:** The scope of an implementation instance is determined by where it is stored. It may be transient (passed as a method argument and garbage collected shortly after) or it may be long-lived (stored in a field of a service or component).
- **Destruction:** An implementation instance is eligible for garbage collection as soon as it is no longer referenced.

**WARNING:** Be mindful of capturing variables from an enclosing scope (creating a closure). If a long-lived object holds a reference to a lambda that captures a short-lived object, it can cause unintentional memory leaks.

## Internal State & Concurrency
- **State:** The TriBoolFunction interface is inherently stateless. However, an *implementation* (a lambda) can become stateful if it captures mutable variables from its enclosing scope. This is a powerful but dangerous feature. Stateful lambdas can introduce side effects and make behavior difficult to reason about.
- **Thread Safety:** The interface contract is thread-safe. The thread safety of an *implementation* is entirely the responsibility of the developer who writes it. If a lambda implementation accesses or modifies shared mutable state, it must employ appropriate synchronization mechanisms (e.g., locks, atomic variables).

**WARNING:** Never assume an arbitrary TriBoolFunction implementation is thread-safe. Treat all implementations as non-thread-safe unless explicitly documented otherwise. Passing stateful, non-synchronized lambdas to multi-threaded systems is a common source of severe concurrency bugs.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(T, U, V, boolean) | R | O(impl) | Executes the function. The complexity is entirely dependent on the specific implementation provided. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to use TriBoolFunction as a parameter in methods that require a flexible, user-defined piece of logic. This inverts control and allows the method to be generic while the caller provides the specific behavior.

```java
// A system that processes entities based on a supplied rule.
public class EntityProcessingSystem {
    public <R> R processEntity(Entity entity, World world, Player player, boolean isPriority, TriBoolFunction<Entity, World, Player, R> rule) {
        // The system doesn't know the rule, it just executes it.
        return rule.apply(entity, world, player, isPriority);
    }
}

// Usage:
EntityProcessingSystem system = ...;
Entity target = ...;

// Provide the behavior via a lambda expression.
String result = system.processEntity(target, world, player, true, (entity, wrld, plyr, isPrio) -> {
    if (isPrio && entity.isHostile()) {
        return "HighPriorityTarget";
    }
    return "NormalTarget";
});
```

### Anti-Patterns (Do NOT do this)
- **Complex Logic in Lambdas:** Avoid writing multi-line, complex business logic directly inside a lambda. If the logic is more than a few lines, refactor it into a private method and use a method reference. This maintains readability and testability.
- **Stateful Lambdas in Concurrent Systems:** Do not use a lambda that captures and modifies external mutable state in a parallel stream or multi-threaded task executor without explicit and correct synchronization. This will lead to race conditions.
- **Ignoring Nulls:** If the return type R can be null, the calling code must be robust against NullPointerExceptions. A better pattern is to use `Optional<R>` as the return type to make the possibility of an absent value explicit in the contract.

## Data Pipeline
TriBoolFunction acts as a discrete, stateless transformation step in a data flow. It does not manage data persistence or transport; it simply defines a computation.

> Flow:
> Caller provides 4 inputs (T, U, V, boolean) -> **TriBoolFunction.apply()** -> Implementation Logic -> Returns 1 output (R) -> Caller receives R.

