---
description: Architectural reference for AbsDensity
---

# AbsDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class AbsDensity extends Density {
```

## Architecture & Concepts
The AbsDensity class is a fundamental component within Hytale's procedural world generation framework. It functions as a *modifier node* within a larger computational graph of Density objects. Its sole purpose is to transform an incoming density value by applying the mathematical absolute value function.

In the context of procedural generation, density functions produce a scalar field (a value for every point in 3D space). Negative values typically represent air or empty space, while positive values represent solid material. By applying an absolute value, AbsDensity can be used to create symmetrical features. For example, a simple gradient that goes from -1 to 1 can be transformed into a V-shape, which is useful for creating ravines, sharp ridges, or symmetrical cave patterns.

This class embodies the Decorator or Chain of Responsibility pattern, where it wraps another Density object (its input) and modifies its output without altering the input's core logic. It is a stateless, purely functional operator within the generation pipeline.

## Lifecycle & Ownership
- **Creation:** Instances of AbsDensity are typically not created directly. They are instantiated by a higher-level world generator service during the construction of a Density graph. This graph is often defined in external configuration files (e.g., JSON or HOCON) and deserialized at server startup or when a world is loaded.
- **Scope:** The lifetime of an AbsDensity instance is tied to the lifetime of the generator graph it belongs to. It is a lightweight object that persists as long as the world's generation rules are held in memory.
- **Destruction:** The object is eligible for garbage collection when the parent Density graph is unloaded or replaced. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The class is stateful, containing a single mutable field: *input*. This field holds a reference to the next Density node in the graph. While the reference itself can be changed via the setInputs method, the object performs no internal caching and its computational logic is stateless.
- **Thread Safety:** **Not guaranteed.** The process method is conditionally thread-safe. It is safe to call from multiple threads only if the underlying *input* Density's process method is also thread-safe. The primary concurrency risk lies with the setInputs method.

   **WARNING:** The setInputs method is a write operation and is **not thread-safe**. Modifying the Density graph by calling setInputs while world generation is active across multiple threads will lead to race conditions, non-deterministic generation, and potential crashes. The graph structure should be considered immutable after its initial construction.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AbsDensity(Density) | constructor | O(1) | Constructs the node with an initial input. |
| process(Context) | double | O(C_input) | Processes the input node and returns the absolute value of its result. Complexity is determined by the input node. |
| setInputs(Density[]) | void | O(1) | Replaces the current input node with the first element of the provided array. Throws ArrayIndexOutOfBoundsException if the array is empty. |

## Integration Patterns

### Standard Usage
AbsDensity is used as a link in a chain of Density nodes to shape the final output of a terrain function. It is composed by a generator builder, not used in isolation.

```java
// Conceptual example of building a generator graph
// A noise function that produces values from -1.0 to 1.0
Density perlinNoise = new PerlinNoiseNode(...);

// Wrap the noise in AbsDensity to create symmetrical valleys/ridges
Density absoluteNoise = new AbsDensity(perlinNoise);

// The generator will then call process on the final node
double value = absoluteNoise.process(generationContext);
```

### Anti-Patterns (Do NOT do this)
- **Dynamic Reconfiguration:** Do not call setInputs on an AbsDensity node after the generation process has begun. The Density graph should be treated as immutable once constructed to ensure deterministic and thread-safe world generation.
- **Null Input:** While the code handles a null input by returning 0.0, this indicates a broken or misconfigured generator graph. A valid graph should never contain nodes with null inputs unless explicitly designed for that purpose (e.g., a root node).
- **Cyclical Dependencies:** Creating a graph where a node is its own ancestor (e.g., `node.setInputs(new Density[]{node})`) will result in a StackOverflowError during the process call.

## Data Pipeline
The data flow for this component is linear and unidirectional during computation. It acts as a simple transformation step in a larger pipeline.

> Flow:
> Generator Engine -> Parent Density Node -> **AbsDensity.process(context)** -> Input Density.process(context)
>
> Return Flow:
> double from Input -> **Math.abs(double)** -> double to Parent Node

