---
description: Architectural reference for FunctionDensity
---

# FunctionDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Node

## Definition
```java
// Signature
public class FunctionDensity extends Density {
```

## Architecture & Concepts
FunctionDensity is a fundamental component within Hytale's procedural world generation framework. It operates as a single-input, single-output processing node within a larger **Density Graph**. Its primary role is to apply an arbitrary mathematical transformation to the density value produced by its input node.

This class embodies a functional, compositional approach to world generation. By chaining these nodes, architects can construct complex procedural logic for shaping terrain. For example, a FunctionDensity node can be used to clamp values, invert a noise field, apply a smoothing curve, or scale the output of another Density node. It acts as a highly flexible adapter or filter in the density processing pipeline.

The core of this node is the `Double2DoubleFunction`, which decouples the graph structure from the specific mathematical operation being performed, allowing for extensive customization and reuse.

### Lifecycle & Ownership
-   **Creation:** FunctionDensity instances are not intended for direct, ad-hoc instantiation. They are created by a higher-level graph builder or deserializer when a world generation graph is loaded from a configuration asset. The specific `Double2DoubleFunction` is injected at this time.
-   **Scope:** The lifetime of a FunctionDensity object is strictly bound to the lifetime of the Density Graph it is a part of. It is a transient object, existing only to fulfill a specific generation task.
-   **Destruction:** The node is eligible for garbage collection as soon as the parent Density Graph is no longer referenced, which typically occurs after a world chunk or region has been fully generated and its density data is baked.

## Internal State & Concurrency
-   **State:** The internal state is **conditionally mutable**.
    -   The transformation logic, represented by the `function` field, is **immutable** after construction.
    -   The connection to the upstream data source, the `input` field, is **mutable**. It can be reassigned after construction via the `setInputs` method, allowing for dynamic graph reconfiguration.
    -   This class does not cache the result of its `process` operation. Each invocation triggers a full re-computation down the input chain.

-   **Thread Safety:** This class is **not thread-safe**. The mutable `input` field makes it vulnerable to race conditions if `setInputs` is called concurrently with a `process` operation. The world generation engine MUST ensure that a given Density Graph is either processed by a single thread at a time or that its structure is not modified during concurrent processing.

## API Surface
The public contract is minimal, focusing exclusively on graph processing and configuration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context context) | double | O(N) | Computes the density by first processing its input node, then applying the internal function. Complexity is dependent on the depth (N) of the input graph. |
| setInputs(Density[] inputs) | void | O(1) | Reconfigures the node's single input. **Warning:** This method only considers the first element of the array; all others are ignored. |

## Integration Patterns

### Standard Usage
FunctionDensity is used to wrap another Density node and transform its output. The function is typically provided as a lambda or method reference during graph construction.

```java
// Assume 'sourceNoise' is a pre-existing Density node (e.g., PerlinNoiseDensity)
// This function will clamp the output of the source noise between 0.2 and 0.8
Double2DoubleFunction clamp = (val) -> Math.max(0.2, Math.min(0.8, val));

// Create the node, linking it to the source
FunctionDensity clampedNoise = new FunctionDensity(clamp, sourceNoise);

// In a real system, this would be part of a larger graph assembly
// double result = clampedNoise.process(context);
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Functions:** The provided `Double2DoubleFunction` must be a pure function. Introducing external state, randomness, or other side effects will violate the determinism of the world generator, leading to inconsistent results and chunk boundary artifacts.
-   **Incorrect Input Configuration:** Calling `setInputs` with an array containing more than one `Density` is misleading and wasteful. Only `inputs[0]` is ever used. An empty array will sever the input connection, causing `process` to always return 0.0.
-   **Processing a Disconnected Node:** Calling `process` after the node has been disconnected (by calling `setInputs` with an empty array) is valid but inefficient, as it will always yield a constant value of 0.0.

## Data Pipeline
FunctionDensity acts as a transformation step in a larger data flow. It receives a single floating-point value and outputs a new one.

> Flow:
> Upstream Density Node.process() → `double` → **FunctionDensity**.function.get() → `transformed double` → Downstream Consumer (e.g., another Density node or the final terrain sculptor)

