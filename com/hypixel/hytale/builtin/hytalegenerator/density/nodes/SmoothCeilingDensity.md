---
description: Architectural reference for SmoothCeilingDensity
---

# SmoothCeilingDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class SmoothCeilingDensity extends Density {
```

## Architecture & Concepts
SmoothCeilingDensity is a computational node within the procedural world generation framework. It functions as a mathematical filter in a directed acyclic graph of Density objects. Its specific role is to apply a "smooth ceiling" to an incoming density value, effectively capping it at a given limit without creating harsh, unnatural transitions.

This class is a key component for shaping terrain and other generated features. Unlike a standard minimum function which produces sharp angles, this node uses a smoothing algorithm (via Calculator.smoothMin) to create gentle curves as the input value approaches the defined limit. This is essential for generating organic-looking plateaus, flattening ocean floors, or limiting the height of geological features.

It embodies the **Composite Pattern**, where it acts as a Density processor that is itself composed of another input Density. This allows for the chaining and nesting of multiple density operations to build complex generation logic.

## Lifecycle & Ownership
- **Creation:** Instantiated during the construction of a world generator's density graph. This is typically driven by a configuration loader or a procedural graph builder, not by core application services.
- **Scope:** The lifetime of a SmoothCeilingDensity instance is tied to its parent world generator configuration. It is not a global or session-scoped object.
- **Destruction:** The object is eligible for garbage collection when the world generator configuration it belongs to is unloaded or replaced. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** The internal state consists of the configuration parameters limit and smoothRange, and a reference to an input Density. While the input reference can be mutated via setInputs, the configuration values are effectively immutable after construction. The class itself holds no dynamically calculated or cached state.

- **Thread Safety:** The process method is conditionally thread-safe. It performs no internal locking and does not modify its own state during execution. Its safety is therefore dependent on the thread safety of the upstream input Density node it calls. The graph structure, however, is **not thread-safe** for modification.

    **WARNING:** Calling setInputs from one thread while another thread is executing the process method on any node in the same graph will lead to race conditions and undefined behavior. The density graph must be treated as immutable during generation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(C_input) | Processes the input Density and applies the smooth ceiling function. Complexity is determined by the input node. Returns 0.0 if no input is configured. |
| setInputs(Density[] inputs) | void | O(1) | Replaces the current input Density with the first element of the provided array. This is a structural mutation. |

## Integration Patterns

### Standard Usage
SmoothCeilingDensity is used as an intermediate node in a density graph to control the upper bounds of a value. It is typically chained after a noise function or another transformation.

```java
// Assume 'perlinNoise' is a configured Density node
// This creates a plateau effect, smoothly capping the noise value at 50.0
double limit = 50.0;
double smoothness = 8.0;
Density plateauNode = new SmoothCeilingDensity(limit, smoothness, perlinNoise);

// In the generator loop:
double finalDensity = plateauNode.process(generationContext);
```

### Anti-Patterns (Do NOT do this)
- **Cyclical Dependencies:** Never construct a graph where a SmoothCeilingDensity node directly or indirectly depends on itself as an input. This will cause a StackOverflowError during the process call.
- **Concurrent Modification:** Do not call setInputs on any node in a density graph while it is being actively processed by the world generator. All graph construction and modification must be completed before parallel processing begins.
- **Negative Smoothness:** The constructor enforces that smoothRange cannot be negative. Attempting to bypass this will result in an IllegalArgumentException.

## Data Pipeline
The class acts as a transformation step in a recursive data flow. Data does not persist within the object; it is processed and passed on immediately.

> Flow:
> World Generator -> `process(context)` -> **SmoothCeilingDensity** -> `input.process(context)` -> (Recursive calls end at a base node like a noise function) -> Base value returned -> **SmoothCeilingDensity applies Calculator.smoothMin** -> Final capped value returned

---

