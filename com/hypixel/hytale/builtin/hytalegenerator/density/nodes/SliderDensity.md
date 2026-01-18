---
description: Architectural reference for SliderDensity
---

# SliderDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Node

## Definition
```java
// Signature
public class SliderDensity extends Density {
```

## Architecture & Concepts
The SliderDensity class is a foundational component within the procedural world generation's *Density Graph* system. It functions as a **Coordinate Space Transformer**, a non-generative node that modifies the input coordinates for its child nodes.

Its primary role is to apply a static three-dimensional translation, or "slide," to the sampling position before it is evaluated by the subsequent density function in the graph. It does not create or modify density values itself; it merely changes *where* the child node is sampled.

This node is a critical building block for spatial composition. It allows designers to reuse a single, complex density function (e.g., a function defining a mountain or a cave system) and place instances of it at various locations within the world without redefining the function itself. It is analogous to a "Translate" or "Offset" operation in a 3D scene graph.

### Lifecycle & Ownership
- **Creation:** Instantiated by the Hytale Generator framework during the parsing and construction of a world generation Density Graph, typically from a data file. It is not intended for manual instantiation during the game loop.
- **Scope:** The object's lifetime is strictly bound to its parent Density Graph. It persists as long as the world generation configuration is active.
- **Destruction:** The object is marked for garbage collection when the entire Density Graph is unloaded or replaced. There is no manual destruction mechanism.

## Internal State & Concurrency
- **State:** The internal state is effectively **immutable** during the density processing phase. The translation vector (slideX, slideY, slideZ) is declared final and set at construction. The child *input* node, while mutable via setInputs, is expected to be configured only once during the graph's initial linking phase. The process method does not mutate any internal fields.

- **Thread Safety:** This class is **conditionally thread-safe**. The process method is re-entrant and free of side effects, making it safe for concurrent execution by multiple world-generation threads.
    - **WARNING:** The thread safety guarantee assumes the graph is fully constructed. Calling setInputs while other threads are calling process will result in undefined behavior and likely cause severe world generation artifacts. All graph mutations must be completed before parallel processing begins.

## API Surface
The public API is minimal, reflecting its role as a simple, data-driven node within a larger system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(1) | Translates the context's position and delegates processing to its child input node. Returns 0.0 if no input is connected. |
| setInputs(Density[] inputs) | void | O(1) | Connects a child node. This is a graph-linking method and is not intended for use after initialization. |

## Integration Patterns

### Standard Usage
This class is not designed for direct invocation by gameplay systems. It is configured within a world generation data file and executed by the Density Graph processor. The following example illustrates how the *engine* would conceptually use the node.

```java
// Engine-level code during world generation
// NOTE: This is a conceptual illustration. Do not use this pattern directly.

// 1. A context is created for a specific world coordinate
Density.Context context = new Density.Context(new Vector3d(100.0, 64.0, 250.0));

// 2. The SliderDensity node is retrieved from the pre-built graph
// This node was configured to slide its input by (50, 0, 50)
SliderDensity slider = densityGraph.getNode(SliderDensity.class);

// 3. The node is processed
// Internally, it creates a new context with position (50, 64, 200)
// and passes it to its child node (e.g., a PerlinNoiseDensity)
double densityValue = slider.process(context);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new SliderDensity()` in gameplay or system logic. World generation functions should be entirely data-driven. Modifying the Density Graph at runtime is an unsupported and hazardous operation.
- **Post-Initialization Mutation:** Never call setInputs on a SliderDensity node after the world generator has begun its processing phase. This will break the immutability assumption and lead to race conditions between worker threads.

## Data Pipeline
The SliderDensity acts as a simple transformation step in the density calculation pipeline. It receives a context, modifies the positional data within it, and forwards it to the next node.

> Flow:
> Density.Context (Original Position) -> **SliderDensity** (Applies vector translation to position) -> Density.Context (New Translated Position) -> Child Density Node -> Final Density Value (double)

