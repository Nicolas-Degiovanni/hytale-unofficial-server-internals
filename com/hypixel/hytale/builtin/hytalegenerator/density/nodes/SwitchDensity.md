---
description: Architectural reference for SwitchDensity
---

# SwitchDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class SwitchDensity extends Density {
```

## Architecture & Concepts
The SwitchDensity class is a conditional routing node within the procedural world generation's density function graph. Its primary architectural role is to act as a multiplexer, selecting one of its several input density functions to execute based on an integer state provided by the processing context.

This component is critical for implementing biome-specific or region-specific terrain variations. Instead of constructing entirely separate and redundant density graphs for each biome, a single graph can use SwitchDensity nodes at key branching points. The context, which carries information about the world coordinate being processed, dictates which branch of the graph is activated. This dramatically simplifies the overall graph structure and centralizes the logic for terrain variation.

For example, a SwitchDensity node could be configured to select a flat noise function when the context's state indicates a "plains" biome, and a complex ridged noise function when the state indicates "mountains".

### Lifecycle & Ownership
-   **Creation:** SwitchDensity nodes are not created in isolation. They are instantiated by a higher-level graph builder, typically during the parsing of a world generation configuration file. They are created as part of a larger, interconnected graph of Density objects.
-   **Scope:** The lifetime of a SwitchDensity instance is strictly bound to the lifetime of the world generation graph it is a part of. These are short-lived objects used for a specific generation task and are not persisted across sessions.
-   **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as the root of its containing Density graph is no longer referenced. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
-   **State:** This class is stateful. It maintains an internal array of input Density nodes and a corresponding immutable array of integer switch states. The `inputs` array is mutable and can be altered after construction via the `setInputs` method.
-   **Thread Safety:** This class is **not thread-safe**. The `process` method reads the internal `inputs` array, while the `setInputs` method writes to it. Concurrent execution of these methods will result in a race condition, leading to unpredictable behavior or runtime exceptions.

    **WARNING:** The entire Density graph, including all its nodes, must be treated as single-threaded during a `process` operation. Any modifications to the graph structure must be performed between processing runs, not during.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context context) | double | O(N) | Selects and processes an input Density node based on the context's switchState. N is the number of configured states. Returns 0.0 if no match is found. |
| setInputs(Density[] inputs) | void | O(M) | Replaces the current set of input nodes. M is the number of inputs. This is a mutable, non-thread-safe operation. |

## Integration Patterns

### Standard Usage
The SwitchDensity node is used to direct the flow of density calculation based on contextual information, such as a biome ID. The context is configured by the caller before initiating the processing of the graph.

```java
// Example: Define two different terrain functions
Density flatTerrain = new ConstantDensity(0.1); // Represents plains
Density mountainTerrain = new PerlinNoiseDensity(...); // Represents mountains

// Configure the SwitchDensity to choose between them based on an integer state
// State 0 -> flatTerrain
// State 1 -> mountainTerrain
SwitchDensity biomeSwitch = new SwitchDensity(
    List.of(flatTerrain, mountainTerrain),
    List.of(0, 1)
);

// --- During world generation ---

// Process a point within the "plains" biome
Density.Context plainsContext = new Density.Context(x, y, z, 0); // switchState = 0
double densityForPlains = biomeSwitch.process(plainsContext);

// Process a point within the "mountains" biome
Density.Context mountainsContext = new Density.Context(x, y, z, 1); // switchState = 1
double densityForMountains = biomeSwitch.process(mountainsContext);
```

### Anti-Patterns (Do NOT do this)
-   **State Mismatch:** Do not construct a SwitchDensity with lists of different sizes for inputs and switch states. This violates the class's contract and will immediately throw an `IllegalArgumentException`. Each input must have a corresponding switch state.
-   **Concurrent Modification:** Do not call `setInputs` from one thread while another thread is calling `process`. This is a severe race condition that will corrupt the state of the generation process. All structural modifications to the density graph must be externally synchronized and completed before any processing begins.

## Data Pipeline
The SwitchDensity acts as a conditional gate in the data flow. It does not generate data itself but rather routes the processing request to a downstream node.

> Flow:
> Density.Context -> **SwitchDensity** (State-based routing) -> Selected Input Density Node -> `double` (Density Value)

