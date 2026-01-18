---
description: Architectural reference for SumDensity
---

# SumDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Component

## Definition
```java
// Signature
public class SumDensity extends Density {
```

## Architecture & Concepts
The SumDensity class is a fundamental component within Hytale's procedural world generation framework. It functions as a mathematical operator node within a larger density function graph. In architectural terms, it embodies the *Composite* design pattern, treating a collection of individual Density objects as a single, unified Density object.

Its primary role is to aggregate the outputs of multiple input density functions by summing their results. This allows for the layering of generative features. For example, a base terrain height (a Density function) can be combined with various noise functions (other Density functions) to create more complex and natural-looking landscapes. SumDensity is the mechanism for this additive blending.

This class is not a standalone service but a building block, intended to be composed into a tree or graph structure that is recursively evaluated by the world generator during chunk creation.

## Lifecycle & Ownership
-   **Creation:** SumDensity instances are typically created during the initialization of a world generator. They are instantiated by a higher-level configuration loader or a builder class that constructs the complete density graph based on world generation presets. They are not intended to be created on-the-fly during the main game loop.
-   **Scope:** The lifetime of a SumDensity object is tied to the world generation graph it belongs to. It persists as long as that generator configuration is active. For a given world, this graph may be created once and used for all subsequent chunk generation tasks.
-   **Destruction:** Instances are managed by the Java Garbage Collector. They are eligible for collection when the world generator that owns their graph is destroyed or reconfigured, and all references to them are released.

## Internal State & Concurrency
-   **State:** The internal state is mutable and consists of a single field: an array of child Density objects named *inputs*. Both the constructor and the setInputs method perform a defensive copy of the incoming collection, preventing external modifications to the original list or array from affecting the internal state of the SumDensity instance.

-   **Thread Safety:** This class is **conditionally thread-safe**. The process method is read-only with respect to its own state. It does not use any locks or synchronization primitives. Its thread safety is therefore entirely dependent on the thread safety of its child *inputs*. If all contained Density nodes are themselves thread-safe, a SumDensity instance can be safely processed by multiple threads simultaneously, which is a critical requirement for parallelized chunk generation.

    **Warning:** Modifying the inputs via setInputs while the process method is being executed by another thread will lead to a race condition and undefined behavior. All mutations to the density graph should be completed before it is used for generation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SumDensity(List<Density> inputs) | Constructor | O(N) | Constructs the node, creating a defensive copy of the provided input list. |
| process(Density.Context context) | double | O(N) | Calculates the sum of the results of calling process on all child nodes. N is the number of inputs. |
| setInputs(Density[] inputs) | void | O(N) | Replaces the internal set of child nodes. Creates a defensive copy of the provided array. |

## Integration Patterns

### Standard Usage
The standard pattern is to compose a SumDensity node as part of a larger density graph during the world generator's setup phase.

```java
// Assume noiseFunction and baseTerrain are pre-configured Density objects
List<Density> layers = new ArrayList<>();
layers.add(noiseFunction);
layers.add(baseTerrain);

// Create the composite node
SumDensity finalTerrainDensity = new SumDensity(layers);

// During chunk generation, the generator provides the context and evaluates the graph
// This call would be deep inside the world generator's logic
double densityValue = finalTerrainDensity.process(generationContext);
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation During Generation:** Never call setInputs on a SumDensity instance that is actively being used by the world generator. The density graph should be treated as immutable once generation begins.

-   **Retaining and Modifying Input List:** Do not modify the original list provided to the constructor with the expectation that the SumDensity instance will see the changes. The constructor makes a copy.

    ```java
    // BAD: This modification will have no effect on combinedDensity
    List<Density> components = new ArrayList<>();
    components.add(noise1);
    SumDensity combinedDensity = new SumDensity(components);
    components.add(noise2); // combinedDensity still only has noise1
    ```

## Data Pipeline
SumDensity acts as an aggregation point within the data flow of the density graph evaluation. It does not originate or terminate a pipeline but rather transforms it.

> Flow:
> [Density Node 1, Density Node 2, ... N] -> **SumDensity.process()** -> Aggregated double value -> (Used by subsequent nodes or the final voxel builder)

