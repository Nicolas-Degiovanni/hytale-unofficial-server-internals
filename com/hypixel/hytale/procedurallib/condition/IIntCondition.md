---
description: Architectural reference for IIntCondition
---

# IIntCondition

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Functional Interface / Contract

## Definition
```java
// Signature
public interface IIntCondition {
```

## Architecture & Concepts
The **IIntCondition** interface is a fundamental contract within the procedural generation library (*procedurallib*). It formalizes the concept of a conditional test against a single integer value, returning a boolean result. This abstraction is a key enabler of the Strategy design pattern, allowing complex procedural logic to be decoupled from the specific conditions that control it.

At its core, this interface allows developers to treat conditional logic as data. Instead of hard-coding checks like `if (height > 100)`, the system can be configured with an implementation of **IIntCondition**. This makes procedural generation rules more modular, reusable, and configurable from external data sources, such as JSON or script files. It is the primary mechanism for controlling branching logic within procedural graphs, such as determining biome placement, feature spawning, or material selection based on inputs like noise values, coordinates, or seeds.

## Lifecycle & Ownership
As an interface, **IIntCondition** itself has no lifecycle. The following pertains to its concrete implementations.

-   **Creation:** Implementations are typically instantiated as lightweight, often stateless, objects. They are created by higher-level systems like a procedural graph parser or a rule engine. It is common to see them defined as lambdas or anonymous inner classes for localized, specific conditions.
-   **Scope:** The lifetime of an **IIntCondition** implementation is tied to its owner, which is typically a configuration object, a procedural graph node, or a rule set. They may be short-lived, existing only for the duration of a single generation task, or they may be part of a long-lived, cached configuration.
-   **Destruction:** Instances are managed by the Java Garbage Collector and are destroyed when their owning object is no longer reachable.

## Internal State & Concurrency
-   **State:** The contract is inherently stateless. Implementations are **strongly expected** to be immutable. Storing mutable state within a condition is a severe anti-pattern that breaks the determinism of the procedural generator.
-   **Thread Safety:** Implementations must be thread-safe. The procedural generation engine may execute different parts of a world in parallel. By adhering to the stateless and immutable design principle, implementations will be inherently thread-safe without requiring explicit locking.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(int var1) | boolean | O(1) | The primary evaluation method. Executes the conditional logic against the input integer. |
| eval(int seed, IntToIntFunction seedFunction) | boolean | O(1) | A default convenience method. Transforms an input seed using the provided function before passing the result to the primary **eval** method. |

## Integration Patterns

### Standard Usage
Implementations are typically provided as lambdas to systems that require conditional logic. This keeps the logic concise and close to where it is used.

```java
// A system that only places a feature if a condition is met.
// The condition here checks if the input value is even.
proceduralRule.setCondition(value -> (value % 2) == 0);

// Using the seed transformation variant
// The condition checks if a transformed seed is positive.
IntToIntFunction hash = (seed) -> someHashingFunction(seed);
proceduralRule.setCondition(seed -> seed > 0, hash);
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Implementations:** Never create a condition that relies on mutable internal state. This will produce non-deterministic and unpredictable results when used in a multi-threaded procedural generator.

    ```java
    // DO NOT DO THIS
    class BadCounterCondition implements IIntCondition {
        private int callCount = 0;
        public boolean eval(int val) {
            // Result depends on how many times this was called before!
            return (callCount++ > 5);
        }
    }
    ```
-   **Expensive Computations:** The **eval** method is expected to be a fast, simple check. It may be called millions of times during world generation. Any expensive calculations, such as noise generation, should be performed *before* calling **eval**, with the result passed in as the integer argument.

## Data Pipeline
The interface acts as a gate or filter in a data flow. It consumes an integer and produces a boolean that directs subsequent logic.

> Flow:
> Noise Generator -> Integer Value -> **IIntCondition.eval()** -> Boolean Result -> Procedural Graph Branching (e.g., Place Tree vs. Place Rock)

