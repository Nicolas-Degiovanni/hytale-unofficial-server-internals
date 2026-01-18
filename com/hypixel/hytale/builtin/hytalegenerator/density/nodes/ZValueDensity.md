---
description: Architectural reference for ZValueDensity
---

# ZValueDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class ZValueDensity extends Density {
```

## Architecture & Concepts
The ZValueDensity class is a foundational "source" node within the procedural world generation framework. It operates as a component in a larger, interconnected system known as a Density Graph. This graph, a Directed Acyclic Graph (DAG), is composed of various Density nodes that collectively define the shape and substance of the game world.

ZValueDensity's role is to inject a raw coordinate value—specifically the vertical Z-coordinate—into the graph's calculations. It serves as a primary input, allowing subsequent nodes in the graph to create features that are sensitive to altitude. For example, it can be used to flatten terrain below a certain Z-level to form oceans, or to change material layers based on height.

Architecturally, it is a terminal node, meaning it has no inputs of its own. This is explicitly defined by its empty implementation of the setInputs method. Its sole responsibility is to sample the Z position from the provided evaluation context and pass that value downstream.

## Lifecycle & Ownership
- **Creation:** ZValueDensity nodes are not instantiated directly in game logic. They are created by a high-level generator service during the parsing of a world generation configuration. This process reads a declarative definition (e.g., from a JSON or HOCON file) and constructs the entire Density Graph in memory.
- **Scope:** The object's lifetime is bound to the Density Graph it is a part of. It is created once when the world generator is initialized and persists as an immutable part of that generator's definition.
- **Destruction:** The instance is eligible for garbage collection only when its parent Density Graph is discarded. This may occur during a server shutdown or when a completely new world generation profile is loaded.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no member fields and its output is purely a function of its input arguments.
- **Thread Safety:** ZValueDensity is inherently **thread-safe**. Its lack of internal state guarantees that concurrent calls to the process method from multiple world generation threads will not interfere with each other. This design is critical for enabling massively parallel terrain generation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(1) | Returns the Z-coordinate from the provided context. Throws NullPointerException if the context or its position field is null. |
| setInputs(Density[] inputs) | void | O(1) | No-op. Fulfills the Density contract but does nothing, as this node accepts no inputs. |

## Integration Patterns

### Standard Usage
This class is never used in isolation. It is always defined as a node within a Density Graph and referenced by other nodes that require altitude information. The generator's execution engine is responsible for invoking its process method during the evaluation of a world chunk.

A conceptual configuration might look like this:

```yaml
# This is a conceptual representation, not actual code.
# A generator might build a graph from a definition like this.

graph:
  nodes:
    - id: z_value
      type: ZValueDensity
    - id: perlin_noise
      type: PerlinNoiseDensity
      # ... other noise params
    - id: final_density
      type: AddDensity
      inputs: [z_value, perlin_noise] # ZValueDensity is used as an input

# The engine would then evaluate the 'final_density' node.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance via `new ZValueDensity()`. The world generator is solely responsible for constructing these nodes from configuration files. Manual creation bypasses the graph construction logic and will result in a disconnected, useless object.
- **Standalone Invocation:** Calling the process method on an instance outside of a generator-driven evaluation loop is meaningless. The Density.Context object is highly specific and must be provided by the generation pipeline.

## Data Pipeline
ZValueDensity acts as a starting point in a data flow. It transforms a positional request from the world generator into a primitive double value that feeds into the rest of the Density Graph.

> Flow:
> World Generator Chunk Request -> Density.Context (containing world position x,y,z) -> **ZValueDensity.process()** -> double (z-value) -> Upstream Consumer Node (e.g., AddDensity, ClampDensity)

