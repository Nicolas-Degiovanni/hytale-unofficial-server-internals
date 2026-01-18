---
description: Architectural reference for MultiplierDensity
---

# MultiplierDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class MultiplierDensity extends Density {
```

## Architecture & Concepts
The MultiplierDensity class is a fundamental component within the procedural world generation engine, specifically operating as a node within a **Density Graph**. A Density Graph is a directed acyclic graph of operations that, when evaluated for a given coordinate, produces a scalar "density" value used to determine terrain, biome placement, and other world features.

This class functions as a **Composite Node**, designed to combine the outputs of multiple child Density nodes. Its sole responsibility is to multiply the density values returned by its inputs. This operation is essential for blending and modulation techniques in procedural generation. For example, a base terrain noise function can be multiplied by a cave noise function; where the cave function returns zero, the resulting product is zero, effectively carving the cave system out of the base terrain.

The implementation includes a critical performance optimization: it short-circuits the calculation. If any input node evaluates to 0.0, the multiplication chain is immediately terminated and 0.0 is returned, preventing unnecessary processing of subsequent nodes in the chain.

## Lifecycle & Ownership
- **Creation:** An instance of MultiplierDensity is typically instantiated by a higher-level configuration loader or a world generator orchestrator. It is created as part of the construction of a complete Density Graph, which defines the logic for a specific world preset. It is never intended to be created in isolation.
- **Scope:** The object's lifetime is bound to the world generation task for which its graph was constructed. It is a short-lived, transient object used to process a specific set of world chunks and is then discarded. It does not persist across sessions or server restarts.
- **Destruction:** Cleanup is managed by the Java Garbage Collector. Once the reference to the root of the Density Graph is released, the MultiplierDensity instance becomes eligible for collection. There are no native resources or explicit destruction methods.

## Internal State & Concurrency
- **State:** The class is **Mutable**. Its internal state consists of an array of child Density nodes, referred to as `inputs`. This array can be replaced at any time after construction via the `setInputs` method. This mutability allows for dynamic reconfiguration of the Density Graph but carries significant concurrency risks.
- **Thread Safety:** This class is **Not Thread-Safe**. The `process` method reads the internal `inputs` array while the `setInputs` method writes to it. Concurrent calls to these two methods on the same instance will lead to unpredictable behavior and severe race conditions.

    **WARNING:** The entire Density Graph must be treated as immutable during the processing phase. All structural modifications must be completed in a single-threaded context before submitting the graph to a multi-threaded world generation worker pool. The `process` method itself is re-entrant and can be safely called by multiple threads simultaneously as long as the object's state is not being mutated.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| MultiplierDensity(List<Density> inputs) | Constructor | O(N) | Constructs the node, converting the input list to an internal array. |
| process(Density.Context context) | double | O(N) | Calculates the product of all child node outputs. N is the number of inputs. Short-circuits if the product becomes zero. |
| setInputs(Density[] inputs) | void | O(1) | Replaces the internal array of child nodes. This is a highly unsafe operation if performed during concurrent processing. |

## Integration Patterns

### Standard Usage
MultiplierDensity is used to combine multiple noise sources or masks. The typical pattern involves constructing several Density nodes and passing them as a list to the MultiplierDensity constructor.

```java
// How a developer should normally use this
// Assume noise1 and noise2 are other initialized Density nodes
List<Density> inputs = new ArrayList<>();
inputs.add(noise1);
inputs.add(noise2);

// Create the multiplier node to combine the inputs
MultiplierDensity multiplier = new MultiplierDensity(inputs);

// Process the node for a given world context
// The result will be noise1.process(ctx) * noise2.process(ctx)
double finalDensity = multiplier.process(context);
```

### Anti-Patterns (Do NOT do this)
- **State Mutation During Processing:** The most critical anti-pattern is modifying the node's inputs while a world generator is actively using the graph. This will cause catastrophic and difficult-to-debug race conditions. The graph must be considered read-only once processing begins.

- **Empty Input Configuration:** While the code defensively handles a zero-input scenario by returning 0.0, this configuration is logically invalid. A multiplier with no inputs serves no purpose and likely indicates an error in the graph construction logic that should be investigated.

## Data Pipeline
As a processing node, MultiplierDensity does not originate or terminate data. It acts as a transformation step within a larger data flow.

> Flow:
> World Generator requests density at coordinate (X, Y, Z) -> Request propagates down the Density Graph -> Child Density Nodes process the coordinate -> **MultiplierDensity** receives multiple double values -> **MultiplierDensity** computes the product -> The result is returned up the graph to the parent node or the World Generator.

