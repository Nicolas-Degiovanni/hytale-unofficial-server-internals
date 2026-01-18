---
description: Architectural reference for OffsetDensity
---

# OffsetDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class OffsetDensity extends Density {
```

## Architecture & Concepts
The OffsetDensity class is a fundamental node within the procedural world generation's density function graph. It operates as a **Decorator** or filter, modifying the output of an upstream density function.

Its primary role is to apply a vertical, height-dependent (Y-axis) offset to a density field. This capability is critical for sculpting complex terrain features that vary with altitude, such as overhangs, stratified rock layers, or floating islands. The behavior of the offset is not fixed; it is defined by a provided function, allowing for limitless procedural variation.

A key architectural feature is its dual-mode operation. When supplied with an input Density, it acts as a modifier in the chain. However, if no input is provided, it functions as a *source node*, generating a base density value derived solely from the world Y-coordinate. This flexibility makes it a versatile and essential building block for defining the foundational shape of the world.

## Lifecycle & Ownership
-   **Creation:** Instantiated by a high-level world generation service during the construction of a density graph. This typically happens when the system parses a world generation profile (e.g., from a JSON definition) and assembles the corresponding node graph in memory.
-   **Scope:** The object's lifetime is ephemeral, scoped strictly to a single world generation task, such as generating the data for a chunk column. It is not a persistent, session-wide service.
-   **Destruction:** It becomes eligible for garbage collection as soon as the density graph it belongs to is no longer needed. This typically occurs immediately after the generation task for a specific world region is complete and the resulting data has been consumed.

## Internal State & Concurrency
-   **State:** The class is stateful, containing references to its input node and the offset function. However, after its initial construction and wiring via setInputs, its state should be considered **effectively immutable**. The core processing logic does not mutate its internal fields.
-   **Thread Safety:** This class is **conditionally thread-safe**. The process method itself contains no locks and performs no writes to shared memory. Its safety is entirely dependent on the thread safety of its dependencies:
    1.  The upstream input Density node must be thread-safe.
    2.  The provided Double2DoubleFunction instance must be pure or otherwise safe for concurrent execution.

    **Warning:** Providing a stateful, non-thread-safe lambda or function object for the offsetFunc will violate the concurrency model of the world generator, leading to severe and difficult-to-debug race conditions and non-deterministic output.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(N) | Calculates the density. If an input node exists, it recursively calls process on the input and adds the Y-based offset. If no input exists, it generates a value from the offset function. Complexity is O(N) where N is the depth of the input chain. |
| setInputs(Density[] inputs) | void | O(1) | Wires the upstream input node. This is part of the graph assembly phase and must not be called during processing. |

## Integration Patterns

### Standard Usage
OffsetDensity is used to wrap an existing density function, such as Perlin noise, to modify its output based on world height.

```java
// Example: Create a simple vertical gradient to add to a base noise function.
// This function will add a small value to the density as Y increases.
Double2DoubleFunction verticalGradient = (y) -> y * 0.05;

// Assume 'baseTerrainNoise' is a pre-existing Density node.
Density terrainWithGradient = new OffsetDensity(verticalGradient, baseTerrainNoise);

// The graph is now ready. During world generation, the 'process' method is called.
double finalDensity = terrainWithGradient.process(currentContext);
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Offset Functions:** Never use an offset function that relies on or modifies external mutable state. This will break determinism and cause severe threading issues.
    ```java
    // DO NOT DO THIS
    int counter = 0;
    Double2DoubleFunction unsafeFunc = (y) -> {
        counter++; // Unsafe mutation from multiple threads
        return y + counter;
    };
    Density badNode = new OffsetDensity(unsafeFunc, null);
    ```
-   **Post-Construction Re-Wiring:** The setInputs method is intended for the initial construction of the density graph. Calling it after the generation process has begun is an unsupported operation and will lead to undefined behavior.

## Data Pipeline
OffsetDensity acts as a transformation step in a larger data flow. It receives a density value, applies its logic, and passes the result to the next consumer in the chain.

> Flow:
> Upstream Density Node.process() -> Input double -> **OffsetDensity** -> (Input double + offsetFunc(context.position.y)) -> Output double -> Downstream Consumer (e.g., Block Placer)

