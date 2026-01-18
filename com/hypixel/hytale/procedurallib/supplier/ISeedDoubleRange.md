---
description: Architectural reference for ISeedDoubleRange
---

# ISeedDoubleRange

**Package:** com.hypixel.hytale.procedurallib.supplier
**Type:** Functional Interface / Strategy

## Definition
```java
// Signature
@FunctionalInterface
public interface ISeedDoubleRange {
```

## Architecture & Concepts
ISeedDoubleRange is a functional interface that defines a core contract within the procedural generation library. It represents a stateless, deterministic transformation function that maps an input double to an output double, influenced by an integer seed.

This interface is a fundamental building block for implementing the **Strategy Pattern** in procedural algorithms. By abstracting the transformation logic, different implementations of ISeedDoubleRange can be injected into noise generators, biome selectors, or feature placement systems to alter their behavior without modifying the core generation code.

For example, one implementation might linearly scale the input value, another might apply a curve, and a third could use the seed to introduce deterministic, pseudo-random jitter. The provided static instance, DIRECT, serves as a default, no-operation (pass-through) strategy.

## Lifecycle & Ownership
As a functional interface, ISeedDoubleRange does not have a traditional object lifecycle. Its lifecycle is entirely dependent on the lifecycle of its concrete implementations, which are typically lambda expressions or method references.

- **Creation:** Implementations are created ad-hoc, often inline as lambda expressions when configuring a procedural generator. They can also be defined as static constants for common, reusable transformations.
- **Scope:** The scope of an ISeedDoubleRange instance is bound to the object that holds a reference to it, such as a generator configuration object. It is typically short-lived and exists only within the context of a specific generation task.
- **Destruction:** Instances are eligible for garbage collection as soon as they are no longer referenced. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The contract of ISeedDoubleRange implies that all implementations **must be stateless**. The transformation should depend solely on the provided seed and value arguments. Capturing mutable state from an enclosing scope (creating a non-pure function) is a severe anti-pattern that will break determinism and thread safety. The provided DIRECT instance is a stateless singleton.
- **Thread Safety:** Implementations **must be unconditionally thread-safe**. Procedural generation tasks are frequently executed across multiple threads to improve performance. Any implementation that is not re-entrant or relies on shared mutable state will cause race conditions, leading to non-deterministic and corrupt world generation. The function should be pure.

## API Surface
The API consists of a single abstract method, defining the functional contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue(int seed, double value) | double | O(1) | Executes the transformation. Implementations are expected to be constant time, but this is not enforced by the contract. |
| DIRECT | ISeedDoubleRange | N/A | A static, shared instance that performs a no-op transformation, returning the input value directly. |

## Integration Patterns

### Standard Usage
ISeedDoubleRange is intended to be used as a lambda expression to supply custom transformation logic to procedural generation algorithms.

```java
// Example: Using a lambda to scale a value by 50%
ISeedDoubleRange scaler = (seed, value) -> value * 0.5;

// The procedural system would then invoke it
double inputNoise = 0.8;
int worldSeed = 12345;
double scaledNoise = scaler.getValue(worldSeed, inputNoise); // Result: 0.4
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Never create a lambda that modifies or depends on external mutable state. This violates the core principles of determinism and thread safety required for procedural generation.

    ```java
    // DO NOT DO THIS
    double factor = 0.5;
    ISeedDoubleRange badScaler = (seed, value) -> {
        factor += 0.1; // Modifies external state, causing non-determinism
        return value * factor;
    };
    ```

- **Ignoring the Seed:** If an implementation involves randomness, it must derive that randomness from the provided seed. Using a non-seeded random source like java.util.Random will produce different worlds from the same seed.

## Data Pipeline
This interface acts as a transformation node within a larger data processing pipeline, typically for procedural content generation.

> Flow:
> Raw Noise Value -> **ISeedDoubleRange (as Mapping Function)** -> Biome Weight or Voxel Density -> Final World Data

