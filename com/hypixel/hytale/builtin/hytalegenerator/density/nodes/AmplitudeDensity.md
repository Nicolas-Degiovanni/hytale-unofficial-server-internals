---
description: Architectural reference for AmplitudeDensity
---

# AmplitudeDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Component

## Definition
```java
// Signature
public class AmplitudeDensity extends Density {
```

## Architecture & Concepts
The AmplitudeDensity class is a fundamental node within the procedural world generation framework. It functions as a **modulator** or **decorator** in a directed acyclic graph of Density objects. Its primary role is to scale the output of an upstream *input* Density function based on a secondary function, the *amplitudeFunc*.

This component is critical for creating vertically-varying terrain features. The amplitudeFunc is evaluated solely on the vertical axis (the *y* coordinate), allowing developers to control the "strength" or "presence" of a noise pattern at different altitudes. For example, it can be used to confine complex cave noise to underground layers or to fade out mountainous features above a certain height.

Architecturally, it embodies the Decorator pattern. It wraps an existing Density object, augmenting its behavior without altering its core interface. This compositional approach allows for the construction of highly complex and layered world generation logic from simple, reusable nodes.

A key performance optimization is built-in: if the amplitude function evaluates to a value near zero for a given y-coordinate, the entire upstream input Density chain is skipped, preventing expensive and unnecessary calculations.

## Lifecycle & Ownership
- **Creation:** AmplitudeDensity nodes are not instantiated directly by the main game loop. They are constructed by a higher-level configuration system, such as a world generator graph builder, which reads procedural generation rules from data files or scripts.
- **Scope:** An instance of AmplitudeDensity exists for the lifetime of the world generator configuration it is a part of. It is effectively a stateless processing node whose configuration persists as long as the world generator is active.
- **Destruction:** The object is managed by the Java Garbage Collector. It is de-referenced and cleaned up when the world generator or the world instance it belongs to is unloaded. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** This class is stateful, holding references to its input Density and its amplitudeFunc. However, this state is intended to be configured once during an initialization phase and then treated as immutable during the processing phase. The setInputs method provides a mechanism for late-binding the graph, but it is not intended for use during active generation.
- **Thread Safety:** **Conditionally Thread-Safe.** The class itself introduces no mutable state during a process call. It can be safely shared and executed by multiple world generation threads *provided that* its dependencies (the input Density and the amplitudeFunc) are also thread-safe. World generation is a highly parallelized task, and this component is designed to support that paradigm.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(C) | Calculates the modulated density. Complexity is dependent on the cost (C) of the input Density chain and the amplitude function. |
| skipInputs(double y) | boolean | O(A) | Determines if the input chain can be skipped. Complexity is dependent on the cost (A) of the amplitude function. |
| setInputs(Density[] inputs) | void | O(1) | Wires the upstream input Density node. **WARNING:** Not safe to call during generation. |

## Integration Patterns

### Standard Usage
AmplitudeDensity is used to wrap another Density node, scaling its output. This is typically done during the construction of the generator's density graph.

```java
// Assume 'caveNoise' is a Density object generating Perlin noise.
// Assume 'verticalFalloff' is a NodeFunction that is 1.0 below y=64 and 0.0 above.

// The caveNoise will now only have an effect below y=64.
Density modulatedCaveNoise = new AmplitudeDensity(verticalFalloff, caveNoise);

// This node is now used in further processing.
double finalDensity = modulatedCaveNoise.process(someContext);
```

### Anti-Patterns (Do NOT do this)
- **Dynamic Re-Wiring:** Do not call setInputs on an AmplitudeDensity node after the world generation process has begun. This method is for initial graph construction only and is not protected by locks, leading to severe race conditions in a multithreaded environment.
- **Unconnected Input:** While the system gracefully handles a null input by returning 0.0, configuring a node this way is indicative of a logical error in the generator graph. An AmplitudeDensity without an input serves no purpose.
- **Ignoring Optimization:** The skipInputs check is a significant performance feature. Bypassing it or implementing custom logic that fails to leverage it can lead to substantial, unnecessary computation in areas where the density would be zero.

## Data Pipeline
The data flow for a single density calculation is linear and transformative. The component acts as a multiplier in the data stream.

> Flow:
> Density.Context -> **AmplitudeDensity.process()** -> [Evaluates amplitudeFunc(y)] -> [Evaluates input.process()] -> **Multiplication** -> Final double value

