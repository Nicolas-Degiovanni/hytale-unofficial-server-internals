---
description: Architectural reference for TriPredicate
---

# TriPredicate

**Package:** com.hypixel.hytale.function.predicate
**Type:** Functional Interface

## Definition
```java
// Signature
public interface TriPredicate<T, R, S> {
```

## Architecture & Concepts
The TriPredicate interface is a core functional primitive within the Hytale engine. It serves as a standardized contract for any operation that requires a boolean test against three distinct input arguments of potentially different types.

Unlike a standard Java Predicate (one argument) or BiPredicate (two arguments), TriPredicate extends this pattern to accommodate more complex conditional logic without requiring the creation of custom, single-use interfaces. It is a fundamental building block for filtering, validation, and decision-making logic in systems where context is derived from three separate sources. For example, it might be used to determine if an *entity* (T) can perform an *action* (R) within a specific *game zone* (S).

This interface promotes a functional programming style, enabling developers to pass behavior as an argument to higher-order functions, leading to more decoupled and testable code.

## Lifecycle & Ownership
As a functional interface, TriPredicate itself does not have a lifecycle. It is a stateless contract. The lifecycle pertains only to the *implementations* of this interface.

- **Creation:** Implementations are typically created ad-hoc as lambda expressions or method references at the call site. They can also be instantiated as dedicated, named classes for complex, reusable logic.
- **Scope:** The scope of a TriPredicate implementation is almost always transient and local to the method in which it is defined. If assigned to a field, its lifetime is bound to the containing object.
- **Destruction:** The implementation object is eligible for garbage collection as soon as it is no longer referenced, which is usually after the calling method completes.

**WARNING:** Avoid storing stateful TriPredicate implementations in long-lived objects unless their lifecycle is carefully managed, as this can lead to memory leaks.

## Internal State & Concurrency
- **State:** The interface itself is stateless. Any implementation of TriPredicate **should be stateless** to align with functional programming principles. Introducing mutable state into a predicate can lead to unpredictable behavior, especially in parallel processing scenarios.
- **Thread Safety:** The contract is inherently thread-safe. However, the thread safety of a specific *implementation* is the sole responsibility of the developer. A stateful implementation must ensure its own synchronization if it is to be used across multiple threads.

**WARNING:** Implementations of TriPredicate should be pure functions. They should not produce side effects (e.g., logging, network calls, modifying external state). Their sole purpose is to return a boolean result based on the provided inputs.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(T var1, R var2, S var3) | boolean | Implementation-Defined | Evaluates this predicate on the given arguments. Returns true if the inputs match the predicate, otherwise false. |

## Integration Patterns

### Standard Usage
The primary and intended usage pattern for TriPredicate is via lambda expressions, which provide a concise and clear way to define the test logic directly where it is needed.

```java
// Example: A system that checks if a player can place a specific block at a location.
// The TriPredicate defines the placement rule.
public boolean canPlaceBlock(Player player, BlockType block, Vector3 position, TriPredicate<Player, BlockType, Vector3> rule) {
    return rule.test(player, block, position);
}

// Usage at the call site:
TriPredicate<Player, BlockType, Vector3> creativeModeRule = (p, b, pos) -> p.isInCreativeMode();
boolean canPlace = canPlaceBlock(somePlayer, someBlock, somePosition, creativeModeRule);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Creating a TriPredicate that modifies its own internal state during the test method is a severe anti-pattern. This breaks referential transparency and makes behavior difficult to reason about.
- **Complex Class Hierarchies:** While possible, creating a deep class hierarchy implementing TriPredicate is often a sign of over-engineering. If the logic is complex, it is better to delegate the work to a separate service or utility class from within a simple lambda.
- **Side-Effects:** The test method must not modify the arguments or any external system state. A predicate's job is to test, not to act.

## Data Pipeline
TriPredicate does not manage a data pipeline itself; rather, it acts as a **conditional gate** or **filter** within a larger data flow. It is a component that receives data and makes a decision that affects the downstream path of that data.

> **Example Flow: Event Processing**
>
> Game Event -> Event Dispatcher -> **TriPredicate Filter** -> (If true) -> Action Handler
>
> In this flow, the TriPredicate might test `(Event, TargetEntity, WorldState)` to decide if an action handler should be invoked.

