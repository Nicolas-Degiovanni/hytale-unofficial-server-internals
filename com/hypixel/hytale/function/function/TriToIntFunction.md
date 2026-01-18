---
description: Architectural reference for TriToIntFunction
---

# TriToIntFunction

**Package:** com.hypixel.hytale.function.function
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface TriToIntFunction<T, U, V> {
```

## Architecture & Concepts
TriToIntFunction is a functional interface that represents a function accepting three arguments of generic types and producing a primitive integer result. This interface is a core component of the engine's functional programming toolkit, designed for performance-critical code paths.

Its primary architectural purpose is to avoid the overhead of autoboxing and unboxing that would occur if a generic `TriFunction<T, U, V, Integer>` were used. By returning a primitive `int`, it eliminates the need to create an Integer wrapper object, reducing memory allocation and garbage collector pressure. This pattern is heavily utilized in systems that perform numerous calculations per frame, such as physics, rendering, or complex game logic, where even minor performance gains are significant at scale.

It serves as a formal contract for lambda expressions and method references, enabling developers to pass behavior as data.

## Lifecycle & Ownership
As an interface, TriToIntFunction itself has no lifecycle. The lifecycle described here pertains to the *implementations* of the interface, typically lambda expressions or anonymous classes.

- **Creation:** Implementations are created at the call site where a lambda expression or method reference is defined. The Java Virtual Machine synthesizes an object that implements this interface.
- **Scope:** The scope of a TriToIntFunction instance is lexical. It is bound to the lifecycle of the object or context that holds a reference to it. It may be a short-lived local variable within a method or a long-lived field in a service.
- **Destruction:** The function object is eligible for garbage collection once all references to it are dropped.

**WARNING:** Be mindful of capturing variables in lambdas (creating closures). If a short-lived lambda captures a reference to a long-lived object, it may prevent that object from being garbage collected, leading to memory leaks.

## Internal State & Concurrency
- **State:** The interface is inherently stateless. However, an implementation (a lambda) can become stateful if it captures non-final variables from its enclosing scope. Such stateful lambdas are known as non-pure functions and introduce side effects.
- **Thread Safety:** The interface contract is thread-safe. The thread safety of a specific *implementation* is not guaranteed and is the responsibility of the developer.

**WARNING:** A lambda that captures and modifies shared, mutable state is **not thread-safe**. Using such an implementation in a concurrent or parallel system without proper synchronization mechanisms will lead to race conditions and unpredictable behavior. Prefer pure, stateless functions for concurrent operations.

## API Surface
The public contract consists of a single abstract method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(T, U, V) | int | Implementation-Defined | Executes the function logic with the provided arguments and returns a primitive integer result. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to use a lambda expression to provide an implementation for a method that accepts a TriToIntFunction. This allows for concise, inline definition of behavior.

```java
// A system that uses the function to calculate a priority score
public int calculatePriority(Entity entity, World world, GameTime time, TriToIntFunction<Entity, World, GameTime> priorityLogic) {
    return priorityLogic.apply(entity, world, time);
}

// Usage at the call site
Entity myEntity = ...;
World currentWorld = ...;
GameTime gameTime = ...;

// Provide a lambda implementation
int priority = calculatePriority(myEntity, currentWorld, gameTime, (entity, world, time) -> {
    if (entity.isHostile()) {
        return 100;
    }
    return time.isNight() ? 50 : 10;
});
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations in Parallel Streams:** Do not use a stateful lambda that modifies external state within a parallel stream or any other concurrent execution context. This is a classic recipe for data corruption.
- **Blocking Operations:** The implementation of `apply` should be fast and non-blocking. Performing I/O, network requests, or taking long-held locks within the function will severely degrade the performance of the calling system.

## Data Pipeline
This interface does not manage a data pipeline itself; rather, it represents a single, discrete processing step *within* a larger pipeline. It is a functional primitive for transformation or computation.

> Flow:
> Input T, Input U, Input V -> **TriToIntFunction.apply()** -> Output (primitive int)

