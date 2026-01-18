---
description: Architectural reference for the ScaleDensity world generation node.
---

# ScaleDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Node

## Definition
```java
// Signature
public class ScaleDensity extends Density {
```

## Architecture & Concepts
The ScaleDensity class is a foundational component within the procedural world generation engine. It functions as a *transform node* within a directed acyclic graph of Density objects, which collectively define the shape and substance of the game world.

Its sole responsibility is to apply a non-uniform spatial scaling transformation to the input coordinates before they are evaluated by a subsequent node in the density graph. This allows world designers to stretch, squash, or compress procedural features, such as noise patterns or biome distributions, along any axis.

Architecturally, it implements a form of the Decorator pattern. It wraps a child Density node (the *input*) and intercepts the `process` call, modifying the coordinate context before delegating the final density calculation. This design makes it a highly composable and reusable element for building complex generation logic.

## Lifecycle & Ownership
- **Creation:** Instances are created by the world generator's graph assembly service. This typically occurs during server startup or when a new world is initialized, based on declarative world generation configuration files. Direct instantiation by gameplay systems is strongly discouraged.
- **Scope:** The lifetime of a ScaleDensity instance is bound to the lifetime of the world generation graph it is a part of. It is created once and persists for the entire server session or until the world generator is reconfigured.
- **Destruction:** The object is eligible for garbage collection when the root of its Density graph is dereferenced. This typically happens upon server shutdown or when a world is unloaded.

## Internal State & Concurrency
- **State:** The object is stateful. The core transformation vector `scale` and the `isInvalid` flag are immutable and established at construction time for performance. However, the reference to the child `input` node is mutable and can be re-wired post-construction via the `setInputs` method.
- **Thread Safety:** This class is **not thread-safe**. While the `process` method operates on a clone of the incoming context to prevent side effects, a call to `setInputs` on one thread while `process` is executing on another can lead to a race condition. The world generation scheduler is responsible for ensuring that the density graph structure is not mutated while chunk generation tasks are in flight. All graph modifications must occur during a "stop-the-world" phase of the generator's lifecycle.

## API Surface
The public API is minimal, focusing exclusively on graph evaluation and composition.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(1) | Scales the position within the context and delegates processing to the child input node. Returns 0.0 if no input is set or if the scale is invalid. |
| setInputs(Density[] inputs) | void | O(1) | Wires the node's output to another Density node. This method only considers the first element of the input array. |

## Integration Patterns

### Standard Usage
ScaleDensity is intended to be used declaratively within a world generation preset. The generator's bootstrap process will construct and link the nodes. The `process` method should only be invoked by a parent Density node or the root of the generation graph.

```java
// Conceptual usage by the generation engine
// NOTE: This code is illustrative and does not represent a real API call.

// Create a noise source
PerlinNoise perlin = new PerlinNoise(...);

// Create a scaler to stretch the noise vertically
ScaleDensity verticalStretch = new ScaleDensity(1.0, 2.0, 1.0, perlin);

// The engine would then call process on the scaler
// to get a vertically-stretched noise value.
double densityValue = verticalStretch.process(generationContext);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ScaleDensity()` in gameplay code. World generation graphs are highly optimized and managed systems. Modifying them at runtime is unsupported and will lead to unpredictable behavior.
- **Concurrent Modification:** Never call `setInputs` on a ScaleDensity node that is part of an active world generator. This will break the immutability assumption required by the multi-threaded chunk generator and cause severe visual artifacts or server crashes.
- **Zero-Value Scaling:** Providing a scale factor of 0 on any axis is a logical error. While the node handles this by returning a default value of 0.0, it effectively nullifies the entire downstream graph, which is almost never the desired outcome. This should be validated at the configuration level.

## Data Pipeline
ScaleDensity acts as a pure transformation step in the data pipeline. It does not generate or combine density values itself; it only alters the spatial context for subsequent nodes.

> Flow:
> Parent Node `process` call -> **ScaleDensity** receives `Context` -> Internal logic clones and scales `Context.position` -> **ScaleDensity** calls `process` on its child `input` with the new `Context` -> Receives `double` result -> Returns `double` result to parent.

