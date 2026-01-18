---
description: Architectural reference for the Combiner class, a stateful accumulator for procedural generation.
---

# Combiner

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.math
**Type:** Transient State Machine

## Definition
```java
// Signature
public class Combiner {
    // Inner classes: Layer, IntersectionPolicy
}
```

## Architecture & Concepts

The Combiner is a foundational component within the procedural world generation framework, designed to calculate a final density or material value at a specific vertical coordinate. It functions as a highly specialized, stateful accumulator that composites multiple layers of influence to produce a single output value.

Its core design revolves around a fluent builder pattern, implemented via the inner **Layer** class. This architecture allows the world generator to start with a base value (e.g., air or stone density) and incrementally add or modify this value based on a series of rules and constraints. Each layer represents a potential feature, such as a biome, ore vein, or cave system, defined by its density, vertical range, and blending properties.

The parent Combiner object maintains the global state for a single point calculation (the target Y coordinate and the accumulated value), while the transient Layer object encapsulates the complex configuration for a single contribution. This separation of concerns keeps the primary API clean while allowing for detailed control over each composited layer. Upon completion, a Layer directly mutates the state of its parent Combiner, making the entire process a forward-only state machine.

## Lifecycle & Ownership

-   **Creation:** A Combiner is instantiated directly via its constructor (`new Combiner(background, y)`) at the beginning of a density calculation for a single voxel column. It is owned exclusively by the generation algorithm that creates it.
-   **Scope:** The lifetime of a Combiner instance is extremely short and is strictly confined to the scope of the method performing the density calculation. It is designed to be created, used, and immediately discarded.
-   **Destruction:** The object is eligible for garbage collection as soon as the final value has been retrieved via `getValue()` and the local reference goes out of scope. There are no external references or cleanup procedures.

## Internal State & Concurrency

-   **State:** The Combiner is fundamentally a mutable object. Its primary purpose is to accumulate state in its internal `value` field. The inner Layer class is also a mutable state machine that tracks its own configuration until it is finalized.
-   **Thread Safety:** **This class is not thread-safe.** It is designed for single-threaded access within a world generation worker. The internal state is mutated without any synchronization primitives (e.g., locks, atomics).

    **WARNING:** Sharing a Combiner instance across multiple threads will lead to race conditions and non-deterministic world generation. Each worker thread *must* create its own unique Combiner instance for each calculation.

## API Surface

The public contract is a fluent interface designed for chaining.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Combiner(background, y) | constructor | O(1) | Creates a new accumulator with a base value at a target Y coordinate. |
| addLayer(density) | Combiner.Layer | O(1) | Begins the definition of a new layer. Returns a fluent builder for the layer. |
| getValue() | double | O(1) | Returns the final, accumulated value after all layers are finished. |
| *Layer*.withLimits(f, c) | Combiner.Layer | O(1) | **Required.** Sets the vertical floor and ceiling for the layer's influence. |
| *Layer*.withPadding(f, c) | Combiner.Layer | O(1) | **Required.** Sets the vertical padding for smooth blending at the edges. |
| *Layer*.withIntersectionPolicy(p, s) | Combiner.Layer | O(1) | **Required.** Defines how the layer's floor and ceiling interact. |
| *Layer*.finishLayer() | Combiner | O(1) | **Terminal.** Finalizes the layer, performs calculations, and applies the result to the parent Combiner. |

## Integration Patterns

### Standard Usage

The Combiner is intended to be used in a single, chained expression. A calculation starts with the constructor, followed by one or more layer definitions. Each layer definition is a chain of configuration calls terminating with `finishLayer`. The final result is retrieved with `getValue`.

```java
// Standard fluent-style composition of two layers.
double background = 0.0;
double targetY = 64.5;

Combiner combiner = new Combiner(background, targetY);

// Add the first layer (e.g., stone)
combiner.addLayer(1.0)
    .withLimits(0.0, 80.0)
    .withPadding(5.0, 10.0)
    .withIntersectionPolicy(Combiner.IntersectionPolicy.MAX_POLICY, 0.2)
    .finishLayer();

// Add a second, overlapping layer (e.g., dirt)
combiner.addLayer(0.8)
    .withLimits(70.0, 90.0)
    .withPadding(4.0, 4.0)
    .withIntersectionPolicy(Combiner.IntersectionPolicy.MAX_POLICY, 0.2)
    .finishLayer();

double finalDensity = combiner.getValue();
```

### Anti-Patterns (Do NOT do this)

-   **Instance Re-use:** Do not cache and re-use a Combiner instance for a different Y coordinate or a new calculation. Its internal state is specific to the initial parameters. Always create a new instance.
-   **Incomplete Layer Configuration:** The API enforces a contract where a layer must be fully configured before it is finished. Calling `finishLayer` before `withLimits`, `withPadding`, and `withIntersectionPolicy` will result in an `IllegalStateException`.
-   **Layer Re-use:** The inner Layer object is a single-use builder. Calling `finishLayer` more than once on the same Layer instance will throw an `IllegalStateException`.

## Data Pipeline

The Combiner is a computational component, not a data-flow system. Its operational flow is a sequence of state mutations driven by the world generator.

> Flow:
> World Generator provides `(background, y)` -> **new Combiner** -> Generator adds layer with `density` -> **Layer Builder** configures `(limits, padding, policy)` -> `finishLayer()` mutates **Combiner.value** -> Generator retrieves `finalValue`

