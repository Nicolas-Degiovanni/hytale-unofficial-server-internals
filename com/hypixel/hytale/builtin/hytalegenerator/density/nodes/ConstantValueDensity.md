---
description: Architectural reference for ConstantValueDensity
---

# ConstantValueDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class ConstantValueDensity extends Density {
```

## Architecture & Concepts
The ConstantValueDensity class is a foundational component within the procedural world generation system. It represents a terminal or leaf node in a density function graph. In procedural generation, complex terrain is often defined by composing mathematical functions, or *Density* nodes, together. This class provides the simplest possible function: one that returns a constant, pre-defined value regardless of the input coordinates or context.

Its primary role is to serve as a static input or a baseline value for more complex density operations. For example, it can represent a solid layer of stone (e.g., a value of 1.0) or a void of air (e.g., a value of -1.0) that other nodes like noise functions can then modify. Its behavior is completely deterministic and context-independent.

## Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level world generator builder or configuration parser. These objects are typically created once during the initialization of a world generation pipeline, based on zone-specific configuration files.
- **Scope:** The object's lifetime is tied to the parent density graph that contains it. It persists as long as the world generator configuration is loaded in memory.
- **Destruction:** Eligible for garbage collection when the world generator unloads its current configuration, for instance, when a server shuts down or a different world is loaded.

## Internal State & Concurrency
- **State:** This object is **Immutable**. Its single state field, *value*, is marked as final and is set exclusively during construction. It performs no caching and holds no other mutable state.
- **Thread Safety:** Inherently **Thread-Safe**. Due to its immutable nature, a single ConstantValueDensity instance can be safely shared and accessed by multiple world generation threads simultaneously without requiring any locks or synchronization primitives. This is a critical property for achieving high-performance, parallelized terrain generation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(1) | Returns the constant density value provided at construction. The input context is completely ignored. |

## Integration Patterns

### Standard Usage
This class is not intended to be used in isolation. It is designed to be composed within a larger graph of Density nodes to form a complete density function.

```java
// How a developer should normally use this

// Define a constant representing a solid material baseline
Density solidBaseline = new ConstantValueDensity(1.0);

// Define a constant representing an air baseline
Density airBaseline = new ConstantValueDensity(-1.0);

// Use in a more complex operation (hypothetical BlendDensity node)
// This might blend between solid and air based on a noise function
Density finalTerrain = new BlendDensity(noiseFunction, solidBaseline, airBaseline);

// The process call will propagate down the graph, eventually calling
// process() on the ConstantValueDensity instances.
double densityAtPoint = finalTerrain.process(someContext);
```

### Anti-Patterns (Do NOT do this)
- **Modification via Reflection:** Do not attempt to modify the internal *value* field after construction. The immutability of this class is a core design contract relied upon by the multi-threaded world generator. Circumventing this will lead to unpredictable and non-deterministic terrain.
- **Instantiation in Loops:** Avoid creating new ConstantValueDensity instances inside a tight loop (e.g., for each block being processed). If a constant value is needed, instantiate it once and reuse the reference.

## Data Pipeline
As a source node, ConstantValueDensity originates data rather than transforming it. Its output is consumed by other nodes in the density graph.

> Flow:
> **ConstantValueDensity** -> `double` value -> Calling Density Node (e.g., AddDensity, BlendDensity) -> Aggregated Density Value

