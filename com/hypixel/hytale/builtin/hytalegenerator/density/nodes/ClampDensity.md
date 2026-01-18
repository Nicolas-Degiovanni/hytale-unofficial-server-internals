---
description: Architectural reference for ClampDensity
---

# ClampDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class ClampDensity extends Density {
```

## Architecture & Concepts
The ClampDensity class is a fundamental component within the procedural world generation engine, specifically operating within a **Density Function Graph**. It functions as a mathematical filter or transformer node. Its sole responsibility is to receive the output of a preceding Density node and constrain, or "clamp," that value to a fixed numerical range.

This operation is critical for several reasons:
*   **Normalization:** It normalizes the output of unbounded noise functions (like Perlin or Simplex noise), ensuring their influence on terrain is predictable and controlled.
*   **Feature Shaping:** It prevents the generation of extreme or invalid terrain features by capping density values that might otherwise result in artifacts like impossibly tall spikes or deep pits.
*   **Domain Control:** It allows designers to define strict boundaries for how different procedural elements can affect the final density field, which ultimately determines the placement of solid matter versus air in the world.

Architecturally, ClampDensity is a node in a directed acyclic graph (DAG). It is composed with other Density nodes to build complex, multi-layered procedural generation logic.

## Lifecycle & Ownership
-   **Creation:** ClampDensity instances are not intended for direct, ad-hoc creation during the game loop. They are instantiated by a higher-level graph construction service, typically during the server's world-generation initialization phase. This process is often driven by deserializing a world generation preset from a configuration file.
-   **Scope:** The object's lifetime is bound to the lifetime of the parent Density Function Graph it belongs to. It is created once when the generator is configured and persists until that generator is discarded, for example, on server shutdown or when a new world with a different ruleset is loaded.
-   **Destruction:** Cleanup is managed by the Java garbage collector. There is no explicit destruction or teardown method. An instance is eligible for collection once the root of its Density Function Graph is no longer referenced.

## Internal State & Concurrency
-   **State:** The object maintains a small, hybrid state. The clamping boundaries, *wallA* and *wallB*, are **immutable** and defined at construction. The reference to the *input* Density node, however, is **mutable** and can be reassigned post-construction via the setInputs method. This design facilitates dynamic graph assembly.

-   **Thread Safety:** This class is **conditionally thread-safe**. The core process method is a pure function with no internal side effects. It can be safely executed by multiple world-generation worker threads simultaneously, *provided the graph structure is not being modified*.

    **WARNING:** The setInputs method is not thread-safe. Calling setInputs while any thread is executing the process method on the same graph will result in a severe race condition and undefined behavior. The entire Density Function Graph must be treated as "effectively immutable" once the generation phase begins. All graph wiring must be completed during a single-threaded initialization phase.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context context) | double | O(N) | Calculates the density from the input node and clamps it. The complexity is dominated by the input node's process method, denoted as N. Returns 0.0 if no input is configured. |
| setInputs(Density[] inputs) | void | O(1) | Re-wires the input for this node. It only considers the first element of the array. Passing an empty array disconnects the input. |

## Integration Patterns

### Standard Usage
ClampDensity is used as a wrapper around another Density node to control its output range. It is typically chained after a noise generator or a mathematical combination of other nodes.

```java
// A graph builder creates a noise source.
Density perlinNoiseSource = new PerlinNoise(seed, frequency);

// The builder then wraps the source in a ClampDensity node to
// ensure its output is strictly between -1.0 and 1.0.
Density clampedNoise = new ClampDensity(-1.0, 1.0, perlinNoiseSource);

// During world generation, the clamped node is processed.
double finalValue = clampedNoise.process(context);
```

### Anti-Patterns (Do NOT do this)
-   **Dynamic Re-wiring:** Do not call setInputs on a ClampDensity instance while it is being actively processed by world generation threads. This will break the immutability assumption of the processing phase and lead to unpredictable terrain generation.
-   **Incorrect Bounds:** While the system allows it, constructing a ClampDensity where *wallA* is greater than *wallB* is a logical error. The underlying Calculator.clamp function will likely collapse all output to one of the bounds, defeating the purpose of the node.
-   **Null Input:** Relying on the default 0.0 return value for a null input is not recommended for final graph construction. All nodes in a production-ready graph should be properly wired. A null input typically signifies an incomplete or misconfigured graph.

## Data Pipeline
ClampDensity acts as a simple, inline processing stage in a larger data flow. It does not buffer or queue data.

> Flow:
> Upstream Density Node -> `process()` -> Raw double value -> **ClampDensity.process()** -> `Calculator.clamp()` -> Clamped double value -> Downstream Consumer (e.g., Voxel Carver, Biome Selector)

