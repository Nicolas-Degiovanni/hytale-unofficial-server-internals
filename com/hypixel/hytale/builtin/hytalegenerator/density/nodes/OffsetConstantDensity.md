---
description: Architectural reference for OffsetConstantDensity
---

# OffsetConstantDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class OffsetConstantDensity extends Density {
```

## Architecture & Concepts
The OffsetConstantDensity class is a fundamental component within the procedural world generation system. It functions as a simple but essential *transformer node* within a larger graph of Density objects. Its sole purpose is to take the output value from a single upstream Density node and add a constant, pre-configured offset to it.

Architecturally, this class embodies the Decorator pattern. It wraps an existing Density object, augmenting its behavior without altering its core interface. In the context of the world generator's directed acyclic graph (DAG), this node allows for fine-tuning and vertical shifting of terrain features. For example, it can be used to uniformly raise or lower the output of a complex Perlin noise function, effectively changing the base ground level or sea level across a biome.

## Lifecycle & Ownership
- **Creation:** Instances are created during the initialization phase of the world generator. They are typically defined declaratively in configuration files (e.g., JSON) that describe the complete density graph for a world or biome. They are not intended to be created dynamically during gameplay.
- **Scope:** The lifetime of an OffsetConstantDensity object is bound to the lifetime of the world generator configuration it belongs to. It persists in memory as long as that generator is active.
- **Destruction:** The object is marked for garbage collection when the entire density graph is discarded. This occurs when a server shuts down, a world is unloaded, or the generator configuration is reloaded.

## Internal State & Concurrency
- **State:** The internal state is **conditionally mutable**. The `offset` value is an immutable `double` set at construction. However, the reference to the upstream `input` Density node is mutable and can be changed at runtime via the `setInputs` method. This design allows for dynamic graph reconfiguration, though such operations are rare and carry significant risks.
- **Thread Safety:** This class is **not thread-safe**. Concurrent calls to `setInputs` and `process` will lead to race conditions and unpredictable generation results. The entire density graph is expected to be treated as a thread-local or immutable structure during chunk generation. All modifications to the graph must be externally synchronized, typically by pausing all generation worker threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(N) | Recursively processes the input node, adds the constant offset to the result, and returns it. N is the depth of the upstream graph. Returns 0.0 if no input is connected. |
| setInputs(Density[] inputs) | void | O(1) | Replaces the current input node with the first element from the provided array. If the array is empty, the input is set to null. |

## Integration Patterns

### Standard Usage
This class is used to wrap another Density node to adjust its final output value. It is a building block for composing more complex density functions.

```java
// Assume 'baseTerrain' is a pre-existing, complex Density node
Density baseTerrain = new PerlinNoiseDensity(...);

// Define an offset to raise the entire terrain feature by 12.0 units
double verticalShift = 12.0;
OffsetConstantDensity elevatedTerrain = new OffsetConstantDensity(verticalShift, baseTerrain);

// The world generator will later invoke process to get the final value
// for a given coordinate.
double finalDensity = elevatedTerrain.process(generationContext);
```

### Anti-Patterns (Do NOT do this)
- **Dynamic Reconfiguration:** Do not call `setInputs` on this node while world generation threads are active. Modifying the density graph during processing is an unsupported operation and will cause severe visual artifacts and data corruption. Graph structures should be considered immutable after initialization.
- **Chaining for Summation:** While technically possible, chaining multiple OffsetConstantDensity nodes to sum several constants is inefficient. If multiple offsets must be added, it is better to pre-calculate the sum and use a single OffsetConstantDensity node with the final value.

## Data Pipeline
This node acts as a simple transformation step in a larger data processing pipeline. It receives a single floating-point value and outputs a single modified value.

> Flow:
> Upstream Density Node -> `process()` -> **OffsetConstantDensity** -> `(input value + offset)` -> Downstream Consumer Node

