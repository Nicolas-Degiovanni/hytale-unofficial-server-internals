---
description: Architectural reference for SmoothFloorDensity
---

# SmoothFloorDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Component

## Definition
```java
// Signature
public class SmoothFloorDensity extends Density {
```

## Architecture & Concepts
The SmoothFloorDensity class is a node within the procedural world generation's density graph. It functions as a mathematical operator that establishes a minimum value, or a "floor", for an incoming density signal.

Its primary purpose is to enforce topographical constraints, such as setting a minimum ground level or sea floor, without creating artificial-looking hard edges. Instead of a sharp clamp (e.g., `Math.max(inputValue, limit)`), it uses a smoothing function to create a gradual, continuous transition as the input value approaches the specified limit. This is essential for generating organic and natural-looking terrain features.

This node is a fundamental building block in the Hytale Generator framework. It is designed to be composed with other Density nodes, forming a complex graph that ultimately defines the shape of the world. It processes a single input Density and transforms its output according to its configured parameters.

## Lifecycle & Ownership
- **Creation:** Instances are created by the Hytale Generator's graph-building service. This typically occurs during server startup or world loading, when the system parses a declarative world generation preset file (e.g., a JSON configuration). It is not intended for direct instantiation by game logic.
- **Scope:** The object's lifetime is bound to the density graph it is a part of. It persists as long as the world generator configuration is active.
- **Destruction:** The instance is marked for garbage collection when its parent density graph is discarded. This happens when a server shuts down, a world is unloaded, or a new world generation preset is loaded that does not include this node.

## Internal State & Concurrency
- **State:** The internal state consists of `limit`, `smoothRange`, and a reference to the `input` Density. This state is configured at creation and is considered **effectively immutable** during generation. The `setInputs` method can mutate the `input` reference, but this is strictly a graph-construction operation. The class holds no cached data from `process` calls.
- **Thread Safety:** The `process` method is **fully thread-safe and re-entrant**. It performs calculations based on its immutable configuration and the provided `Density.Context` without modifying any internal fields. This design is critical for the world generator, which heavily parallelizes chunk generation across multiple worker threads. All threads can safely share and execute on a single instance of the density graph.

**WARNING:** The `setInputs` method is **not thread-safe**. It mutates the node's internal graph connection. Invoking it while generator threads are active will cause severe race conditions and non-deterministic output. It must only be called from a single thread during the initial graph assembly phase.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(N) | Calculates the density value. Complexity is dominated by the recursive call to the input node's `process` method (N). The local calculation is O(1). |
| setInputs(Density[] inputs) | void | O(1) | Connects this node to its upstream dependency. Throws `ArrayIndexOutOfBoundsException` if the array is non-empty but contains a null element. |

## Integration Patterns

### Standard Usage
Direct interaction with this class in Java is rare. Its behavior is typically defined declaratively within a world generation preset file. The framework then materializes the Java object graph from this configuration.

A conceptual configuration might look like this:
```json
// Hypothetical worldgen preset
{
  "id": "terrain_base_density",
  "type": "smooth_floor",
  "limit": -20.0,
  "smoothRange": 8.0,
  "input": {
    "type": "perlin_noise",
    "scale": 256,
    "octaves": 4
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new SmoothFloorDensity()`. The node must be created and managed by the generator framework to ensure it is correctly integrated into the density graph and dependency injection system.
- **Dynamic Reconfiguration:** Do not call `setInputs` on a node after the world generation process has started. The density graph is assumed to be static once processing begins.

## Data Pipeline
The class acts as a single step in a larger, recursive data processing pipeline. When its `process` method is called, it first triggers the evaluation of its entire input chain and then transforms the final result.

> Flow:
> World Generator Request -> **SmoothFloorDensity.process(ctx)** -> `input.process(ctx)` -> ... -> (Base Noise Function) -> `double` return value -> **Calculator.smoothMax(range, value, limit)** -> Final `double` Density Value

