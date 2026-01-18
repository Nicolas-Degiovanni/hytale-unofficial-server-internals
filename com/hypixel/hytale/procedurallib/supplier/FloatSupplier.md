---
description: Architectural reference for FloatSupplier
---

# FloatSupplier

**Package:** com.hypixel.hytale.procedurallib.supplier
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface FloatSupplier {
   float getAsFloat();
}
```

## Architecture & Concepts
The FloatSupplier interface is a fundamental contract within the procedural generation library. It adheres to the Supplier design pattern, abstracting the source of a primitive float value. Its primary architectural purpose is to decouple procedural algorithms from the concrete data sources they operate on.

Instead of passing a static float value to a function, developers pass an implementation of FloatSupplier. This enables powerful techniques such as:
- **Lazy Evaluation:** The float value is not computed until the getAsFloat method is explicitly invoked. This is critical for performance in complex generation pipelines where a value may not always be needed.
- **Dynamic Values:** The supplied float can change over time or be dependent on other runtime parameters (e.g., world seed, biome type, game state) without altering the consuming algorithm.
- **Composition:** Suppliers can be chained or decorated to create complex, configurable behaviors from simple, reusable components.

This interface is a core primitive for building flexible and data-driven procedural systems, from terrain generation to entity attribute calculation.

## Lifecycle & Ownership
As a functional interface, FloatSupplier itself has no lifecycle. The lifecycle and ownership semantics apply to the *implementing instance*, which is typically a lambda expression or a dedicated class.

- **Creation:** Instances are created ad-hoc, often as lambda expressions passed directly into method calls or stored as fields within configuration objects. For example, `() -> 0.5f` or `() -> noise.sample(x, y)`.
- **Scope:** The scope is determined by what captures the instance. If a lambda is passed to a short-lived procedural carver, its lifetime is transient. If it is stored in a long-lived Biome configuration object, it will persist as long as that configuration is held in memory.
- **Destruction:** The Java Garbage Collector reclaims implementing instances when they are no longer referenced. There is no manual destruction mechanism.

## Internal State & Concurrency
The interface contract is stateless. However, implementations can be stateful, and this has significant implications for concurrency.

- **State:**
    - **Stateless Implementations:** A supplier returning a constant value, like `() -> 100.0f`, is stateless and inherently pure.
    - **Stateful Implementations:** A supplier that reads from a mutable field, queries a non-thread-safe object, or maintains an internal counter is stateful. Example: `() -> this.currentHeightValue`.

- **Thread Safety:** **Not guaranteed.** Thread safety is entirely the responsibility of the implementation. If a FloatSupplier instance is shared across multiple threads (e.g., in a parallelized world generation task), its implementation must be thread-safe. Accessing shared mutable state without proper synchronization within getAsFloat will lead to race conditions and non-deterministic behavior.

**WARNING:** Assume all FloatSupplier implementations are **not** thread-safe unless explicitly documented otherwise by the provider.

## API Surface
The API consists of a single method, defining it as a functional interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAsFloat() | float | Varies | Retrieves the supplied float value. Complexity is entirely dependent on the implementation. |

## Integration Patterns

### Standard Usage
FloatSupplier is used to provide configurable, dynamic values to procedural algorithms. It is most commonly implemented using a lambda expression for brevity.

```java
// A terrain generator that accepts a supplier for mountain height.
// This decouples the generator from how height is determined.
public void generateTerrain(FloatSupplier mountainHeightSupplier) {
    // The value is only computed when needed (lazy evaluation).
    float height = mountainHeightSupplier.getAsFloat();
    // ... use height to generate terrain ...
}

// Usage:
NoiseModule noise = new PerlinNoise(...);
generateTerrain(() -> 150.0f); // Using a constant value
generateTerrain(() -> noise.get(x, z) * 50.0f + 100.0f); // Using a dynamic noise value
```

### Anti-Patterns (Do NOT do this)
- **Complex Internal Logic:** Avoid placing heavy, multi-step computations or I/O operations inside the getAsFloat implementation. This can create severe performance bottlenecks if the supplier is called within a tight loop (e.g., per-voxel). The supplier should be a lightweight accessor.
- **Premature Evaluation:** Calling getAsFloat and caching the result defeats the purpose of lazy evaluation and dynamic value provision. Only call it at the point where the value is actually required.
- **Unsafe Concurrent Sharing:** Do not share a stateful, non-synchronized FloatSupplier implementation across multiple threads. This is a common source of bugs in parallel procedural generation.

## Data Pipeline
FloatSupplier typically acts as a data *source* or the starting point of a data flow within a larger system.

> Flow:
> **FloatSupplier Implementation** (e.g., Lambda, NoiseSampler) -> Invocation of getAsFloat() -> `float` primitive -> Consuming System (e.g., Voxel Carver, Biome Placer, Material Assigner)

