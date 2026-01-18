---
description: Architectural reference for BiToFloatFunction
---

# BiToFloatFunction

**Package:** com.hypixel.hytale.function.function
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface BiToFloatFunction<T, V> {
   float applyAsFloat(T var1, V var2);
}
```

## Architecture & Concepts
BiToFloatFunction is a specialized functional interface that serves as a high-performance contract for functions that accept two generic arguments and produce a primitive float. It is a core component of the engine's functional programming toolkit, designed explicitly to avoid the performance overhead associated with autoboxing and unboxing standard Java wrapper types like Float.

In performance-critical systems such as physics calculations, rendering pipelines, or entity attribute modifiers, the allocation of wrapper objects can lead to significant garbage collector pressure and performance degradation. By operating directly on the primitive float type, this interface ensures that implementing functions can be inlined by the JIT compiler and execute with minimal overhead. It is the canonical choice for representing any two-argument function that results in a float value.

## Lifecycle & Ownership
As an interface, BiToFloatFunction does not have a lifecycle itself. Instead, the lifecycle pertains to its *implementations*, which are typically provided as lambda expressions or method references.

- **Creation:** Implementations are defined inline at the call site where a method requires a BiToFloatFunction. They are instantiated by the Java runtime when the consuming method is invoked.
- **Scope:** The scope of an implementation is typically transient and ephemeral, bound to the execution of the method that accepts it. If a lambda is assigned to a field, its lifetime is extended to match the lifetime of the containing object.
- **Destruction:** Implementations are managed entirely by the Java Garbage Collector. Stateless lambdas have a minimal memory footprint and are eligible for collection as soon as they are no longer referenced.

## Internal State & Concurrency
- **State:** The interface contract is stateless. Implementations should, by convention, be pure functions—meaning their output depends solely on their input arguments, without reliance on or modification of external state. While it is possible for a lambda to close over and capture variables from its enclosing scope, this practice should be approached with extreme caution.
- **Thread Safety:** The interface is inherently thread-safe. The thread safety of a specific *implementation* is the responsibility of the developer.
    - **WARNING:** Stateless implementations (pure functions) are unconditionally thread-safe and are the strongly preferred pattern.
    - **WARNING:** Implementations that capture and modify shared, mutable state are **not** thread-safe and must be protected by explicit synchronization mechanisms. Such stateful implementations are considered an anti-pattern.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| applyAsFloat(T, V) | float | O(N) | Executes the function. Complexity is determined by the specific implementation (N), not the interface itself. |

## Integration Patterns

### Standard Usage
This interface is intended to be consumed by higher-order functions that need to parameterize an operation involving two objects and a float result. The caller provides the logic via a lambda expression.

```java
// A system method that accepts the functional interface
public float calculateInteraction(Entity a, Entity b, BiToFloatFunction<Entity, Entity> damageFunction) {
    // ... pre-calculation logic
    return damageFunction.applyAsFloat(a, b);
}

// Calling the method with a lambda implementation
Entity player = ...;
Entity monster = ...;
float finalDamage = calculateInteraction(player, monster, (p, m) -> {
    float baseDamage = p.getStrength() * 1.5f;
    float defenseModifier = 1.0f - m.getArmorValue();
    return baseDamage * defenseModifier;
});
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Avoid writing lambdas that modify external state. This breaks referential transparency, makes code difficult to reason about, and is a primary source of concurrency bugs.
- **Complex Logic:** Do not embed multi-stage, complex algorithms directly within a lambda. If an implementation exceeds a few expressions, refactor it into a dedicated, well-named private or static method and provide a method reference instead. This improves readability and testability.
- **Using Standard BiFunction:** Never use a generic `BiFunction<T, V, Float>` in performance-sensitive code. The resulting boxing of the primitive float into a Float object will degrade performance and is explicitly what this interface is designed to prevent.

## Data Pipeline
BiToFloatFunction is not a data pipeline stage itself, but rather a building block used to define logic *within* a pipeline. It represents a transformation or calculation step.

> Flow:
> Input A (Type T), Input B (Type V) -> **Implementation of BiToFloatFunction** -> Output (primitive float) -> Consuming System (e.g., Physics Engine, Combat Logic)

