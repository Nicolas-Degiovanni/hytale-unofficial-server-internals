---
description: Architectural reference for CurveMapperDensity
---

# CurveMapperDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Component/Node

## Definition
```java
// Signature
public class CurveMapperDensity extends Density {
```

## Architecture & Concepts
The CurveMapperDensity class is a fundamental transformation node within the procedural world generation framework. It operates within a directed acyclic graph (DAG) of Density objects, where each node contributes to the final density value at a specific 3D coordinate.

The primary role of this class is to remap a numerical input to a new output value by applying a mathematical function. It acts as a filter or decorator, taking the output from a preceding Density node in the graph and passing it through a configurable `Double2DoubleFunction`. This allows for powerful and non-linear shaping of terrain features. For example, it can be used to sharpen noise functions, create plateaus with a step function, or invert terrain by applying a function like `f(x) = 1.0 - x`.

It is a single-input, single-output node. Its behavior is defined entirely by its input node and its configured curve function.

### Lifecycle & Ownership
- **Creation:** Instances are typically created by a world generator's configuration loader. The generator constructs the entire Density graph from a definition file (e.g., JSON), wiring nodes like CurveMapperDensity together. The specific `Double2DoubleFunction` is injected during this construction phase.
- **Scope:** The lifetime of a CurveMapperDensity instance is bound to the world generation task. It is created, used to generate a set of chunks, and then becomes eligible for garbage collection. It is not a persistent, engine-level service.
- **Destruction:** The object is destroyed by the Java garbage collector when the Density graph it belongs to is no longer referenced.

## Internal State & Concurrency
- **State:** The class holds mutable state. The `curveFunction` is immutable and fixed upon construction. However, the `input` Density node can be dynamically replaced at runtime via the `setInputs` method. The class does not perform any internal caching of processed values.
- **Thread Safety:** This class is **not thread-safe**. The `input` field is accessed and written without synchronization. If one thread calls `setInputs` while another is executing `process`, the behavior is undefined and may result in inconsistent world generation. The parent world generation system is responsible for ensuring that the structure of the Density graph is not mutated while it is being actively processed. All modifications to the graph must be externally synchronized or performed between processing tasks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context context) | double | O(N) | Recursively processes its input node, then applies the internal curve function to the result. N is the depth of the input graph. |
| setInputs(Density[] inputs) | void | O(1) | Replaces the current input node with the first element from the provided array. If the array is empty, the input is set to null. |

## Integration Patterns

### Standard Usage
CurveMapperDensity is used to modify the output of another Density node. The standard pattern involves creating a source Density, defining a curve function, and constructing the CurveMapperDensity to wrap the source.

```java
// Example: Create a curve that squares the input, sharpening it.
Double2DoubleFunction sharpeningCurve = (input) -> input * input;

// Assume 'sourceNoise' is a pre-existing Density node (e.g., PerlinNoiseDensity)
Density sourceNoise = ...;

// Create the mapper to apply the curve to the noise output.
CurveMapperDensity sharpenedNoise = new CurveMapperDensity(sharpeningCurve, sourceNoise);

// When the graph is processed, sharpenedNoise will return the square of sourceNoise's value.
double finalValue = sharpenedNoise.process(context);
```

### Anti-Patterns (Do NOT do this)
- **Graph Mutation During Processing:** Modifying the graph via `setInputs` while a world generation worker thread is calling `process` on any node in that same graph is a severe race condition. All graph configuration should be completed before processing begins.
- **Incorrect Input Array:** The `setInputs` method expects an array of length 1. Passing a larger array will cause subsequent inputs to be ignored. Passing an empty array will nullify the input, causing `process` to operate on a default value of 0.0, which is often an undesirable outcome.

## Data Pipeline
The class functions as a simple, stateless transformation step in a larger data pipeline.

> Flow:
> Upstream Density Node -> `process()` -> **double** (raw value) -> **CurveMapperDensity** -> `curveFunction.applyAsDouble()` -> **double** (mapped value) -> Downstream Density Node or Final Output

