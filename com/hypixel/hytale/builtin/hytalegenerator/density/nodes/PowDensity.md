---
description: Architectural reference for PowDensity
---

# PowDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Component

## Definition
```java
// Signature
public class PowDensity extends Density {
```

## Architecture & Concepts
The PowDensity class is a mathematical operator node within the procedural world generation framework. It functions as a single-input, single-output component in a larger Directed Acyclic Graph (DAG) of Density objects. Its sole purpose is to apply an exponentiation function to the value produced by its upstream input node.

This class is a fundamental building block for shaping and transforming density fields, which are used to define terrain, caves, and other world features. For example, applying an exponent greater than 1.0 can sharpen transitions in a noise field, while an exponent less than 1.0 can soften them.

The core contract is defined by the abstract Density base class, which requires a `process` method. PowDensity provides a specific implementation of this contract, acting as a stateless transformer in the data pipeline. It correctly handles negative input values by preserving their sign before applying the power function, preventing domain errors and ensuring symmetrical transformations around zero.

## Lifecycle & Ownership
- **Creation:** PowDensity instances are not intended for direct, standalone instantiation by end-users. They are created by a higher-level graph builder or configuration loader, typically by deserializing a world generation preset.
- **Scope:** The lifetime of a PowDensity object is strictly bound to the lifetime of the generator graph it is a part of. It is a value-like object that exists only to define a step in a larger calculation.
- **Destruction:** Cleanup is managed by the Java Garbage Collector. When the parent generator graph is dereferenced and collected, its constituent nodes, including PowDensity instances, are also collected. There is no explicit destruction or resource release method.

## Internal State & Concurrency
- **State:** The object holds two pieces of state: the `exponent` and the `input` Density.
    - The `exponent` is immutable and is fixed at creation time.
    - The `input` reference is mutable and can be reconfigured via the `setInputs` method. This mutability is designed *only* for the initial graph construction phase.
- **Thread Safety:** The class is not fully thread-safe, but is designed for safe concurrent execution under specific conditions.
    - The `process` method is reentrant and thread-safe. It does not modify any internal state and its computation is deterministic based on its inputs. Multiple threads can call `process` concurrently without issue.
    - The `setInputs` method is **not thread-safe**. Modifying the input node while the `process` method is being executed in another thread will lead to race conditions and non-deterministic output.

**WARNING:** All structural modifications to the Density graph, including calls to `setInputs`, must be completed in a single-threaded context before the graph is used for concurrent world generation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(N) | Processes the input Density node and applies the power function. Complexity is O(N) where N is the complexity of the entire upstream graph. The operation itself is O(1). |
| setInputs(Density[] inputs) | void | O(1) | Sets the upstream input node. **WARNING:** This method is not thread-safe and should only be used during graph initialization. It only considers the first element of the array. |

## Integration Patterns

### Standard Usage
PowDensity is used as an intermediate node to modify the output of another Density function, such as a noise generator.

```java
// Assume 'noise' is a pre-existing Density node (e.g., PerlinNoiseDensity)
// and 'context' is a valid Density.Context for a world location.

// Create a node to sharpen the output of the noise function
double exponent = 2.0;
PowDensity sharpener = new PowDensity(exponent, noise);

// The result will be the noise value raised to the power of 2
double sharpenedValue = sharpener.process(context);
```

### Anti-Patterns (Do NOT do this)
- **State Mutation During Processing:** Never call `setInputs` on a PowDensity instance that is actively being used by the world generator across multiple threads. This will break generation guarantees.
- **Misunderstanding Input Array:** The `setInputs` method accepts an array for interface compatibility, but it only ever uses `inputs[0]`. Passing multiple nodes is misleading and has no effect.
- **Null Input Processing:** While the class gracefully handles a null input by returning 0.0, a properly configured graph should never contain nodes with unlinked inputs. This typically indicates a configuration error.

## Data Pipeline
PowDensity acts as a transformation step in a larger data flow. It receives a single floating-point value and outputs a single transformed value.

> Flow:
> Upstream Density Node -> `process()` -> double value -> **PowDensity.process()** -> `Math.pow()` -> transformed double value -> Downstream Consumer (e.g., another Density node or the final terrain composer)

