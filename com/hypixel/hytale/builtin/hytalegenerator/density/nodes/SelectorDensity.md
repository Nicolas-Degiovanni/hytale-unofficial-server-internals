---
description: Architectural reference for SelectorDensity
---

# SelectorDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Component

## Definition
```java
// Signature
public class SelectorDensity extends Density {
```

## Architecture & Concepts
The SelectorDensity class is a fundamental signal processing node within the procedural world generation framework. It operates as a function in a directed acyclic graph of Density nodes, where each node contributes to the final density value at a given world coordinate.

Its specific role is to **remap and constrain** an incoming density value. It takes the output from a single upstream Density node, normalizes it from a source range to a target range, and then applies either a hard clamp or a smooth, curved clamp to the result.

This component is critical for shaping terrain features. For example, it can take a noisy Perlin noise input (ranging from -1.0 to 1.0) and isolate a specific band of values (e.g., 0.3 to 0.5) to define the height and smoothness of a plateau. The smoothing parameter allows for the creation of soft, natural transitions rather than sharp, artificial cliffs.

## Lifecycle & Ownership
- **Creation:** SelectorDensity nodes are not intended for direct manual instantiation in game logic. They are typically instantiated by a higher-level world generator service that parses a declarative configuration (e.g., a JSON definition) for a specific world or biome. The graph of Density nodes is constructed during this parsing phase.
- **Scope:** The lifetime of a SelectorDensity instance is bound to the world generator configuration it is a part of. It persists in memory as long as that generator is active and used for chunk generation.
- **Destruction:** The object is eligible for garbage collection once the world generator configuration is unloaded or replaced. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The component is stateful, but its state is primarily immutable configuration (`fromMin`, `fromMax`, `toMin`, `toMax`, `smoothRange`) set at construction time. The single mutable field is `input`, which references the upstream Density node. This field is set via the `setInputs` method, typically only once during the initial graph-wiring phase. The `process` method itself is stateless and does not cache results.

- **Thread Safety:** **Conditionally Thread-Safe.** The `process` method is safe to be called from multiple threads simultaneously, as is common in parallel chunk generation. This safety relies on the critical assumption that the Density graph is treated as immutable after its initial construction. The `setInputs` method is a write operation and is **not thread-safe**.

    **WARNING:** Calling `setInputs` on any node in the graph while other threads are calling `process` will lead to race conditions, non-deterministic world generation, and visible chunk seams. The entire generator graph must be fully constructed and wired before being used by worker threads.

## API Surface
The public contract is minimal, focusing on graph construction and processing.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context context) | double | O(D) | Processes the input node and applies the normalization and clamping logic. Complexity is dependent on the depth (D) of the input node's own graph. |
| setInputs(Density[] inputs) | void | O(1) | Wires the upstream node. Expects an array with at least one element, but only uses the element at index 0. |

## Integration Patterns

### Standard Usage
SelectorDensity is used as an intermediate node in a chain. A developer defines its parameters in a configuration file, and the framework wires it between other nodes, such as a noise source and a final output.

```java
// Conceptual example of framework-level wiring

// 1. Create the source noise node
PerlinNoiseDensity perlin = new PerlinNoiseDensity(/* seed, frequency, etc. */);

// 2. Create the Selector to isolate a value range and smooth it
// This will take values from -0.2 to 0.2 from the perlin noise
// and remap them to a 0.0 to 1.0 range with a gentle slope.
SelectorDensity plateauShaper = new SelectorDensity(-0.2, 0.2, 0.0, 1.0, 0.1, null);

// 3. Wire the nodes together
plateauShaper.setInputs(new Density[]{ perlin });

// 4. The generator can now process the final node
// double finalValue = plateauShaper.process(context);
```

### Anti-Patterns (Do NOT do this)
- **Dynamic Re-Wiring:** Do not call `setInputs` after the world generation process has started. The graph should be considered immutable once generation begins to ensure deterministic and thread-safe output.
- **Invalid Ranges:** The constructor validates that `min` is not greater than `max`. Do not attempt to bypass this or create configurations with inverted ranges, as it violates the logical contract of the component.
- **Ignoring Input:** While the `input` field is nullable, using a SelectorDensity with a null input is a logical error. It will always process a value of 0.0, making its normalization logic ineffective. It should always be connected to an upstream node.

## Data Pipeline
The flow of data through this component during a single `process` call is linear and synchronous.

> Flow:
> Density.Context -> `input.process()` -> Raw `double` value -> **SelectorDensity** (Normalization -> Clamping/Smoothing) -> Final `double` value -> Consuming Node or System

