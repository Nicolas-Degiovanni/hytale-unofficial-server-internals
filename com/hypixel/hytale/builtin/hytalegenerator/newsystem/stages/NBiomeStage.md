---
description: Architectural reference for NBiomeStage
---

# NBiomeStage

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.stages
**Type:** Transient

## Definition
```java
// Signature
public class NBiomeStage implements NStage {
```

## Architecture & Concepts
The **NBiomeStage** is a foundational component within the Hytale world generation system. It operates as a concrete implementation of the **NStage** interface, representing a single, discrete step in a larger, multi-stage generation pipeline.

Its specific role is to act as a *source stage* for biome data. Unlike other stages that might transform existing data, **NBiomeStage** generates biome information from scratch for a given world region. It does not consume any input data buffers from preceding stages, making it a common starting point in a generation graph.

Architecturally, this class employs the **Strategy Pattern**. The core biome generation logic is not hardcoded within the stage itself. Instead, it is injected via a **BiCarta** functional interface during construction. This powerful design decouples the pipeline execution framework (**NBiomeStage**) from the specific biome generation algorithm (**BiCarta**), allowing different algorithms to be swapped in without altering the stage's structure.

## Lifecycle & Ownership
-   **Creation:** An **NBiomeStage** instance is created and configured by a higher-level orchestrator, such as a pipeline builder or world generator configuration service. It is never instantiated directly by game logic. The creator must provide the stage name, the definition of the output buffer, and the **BiCarta** implementation containing the generation algorithm.
-   **Scope:** The object's lifetime is tied to the world generator's configuration. It is effectively a stateless, reusable component that persists as long as the generator pipeline it belongs to is active.
-   **Destruction:** The instance is eligible for garbage collection when the world generator configuration is unloaded, typically during a server shutdown or a transition to a new world with a different ruleset.

## Internal State & Concurrency
-   **State:** The internal state of **NBiomeStage** is established at construction and is **effectively immutable**. The fields *stageName*, *biomeOutputBufferType*, and *biomeCarta* are set once and are not modified during the object's lifetime. The class holds no mutable state related to the execution of a specific generation task.

-   **Thread Safety:** This class is **conditionally thread-safe**. Its own state is immutable and safe for concurrent access. Thread safety of the overall operation depends entirely on the injected **BiCarta** implementation. The world generation framework is designed to execute stages in parallel across multiple worker threads. Therefore, any provided **BiCarta** **must** be thread-safe and free of side effects to prevent data corruption and race conditions. The `run` method operates on a thread-local buffer view provided by the context, ensuring that concurrent runs do not directly conflict at the buffer access level.

## API Surface
The public contract is defined by the **NStage** interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| run(NStage.Context context) | void | O(N*M) | Executes the biome generation logic for the region defined in the context. N and M are the width and depth of the region. This is the primary entry point for the stage executor. |
| getInputTypesAndBounds_bufferGrid() | Map | O(1) | Declares data dependencies. **WARNING:** Always returns an empty map, signifying this is a source stage with no inputs. |
| getOutputTypes() | List | O(1) | Declares the data buffer this stage produces. |
| getName() | String | O(1) | Returns the unique identifier for this stage instance. |

## Integration Patterns

### Standard Usage
Developers do not invoke **NBiomeStage** methods directly. Instead, they configure an instance and register it with the world generator's pipeline orchestrator. The system then manages its execution.

```java
// In a world generator configuration file or builder:

// 1. Define the biome generation algorithm (the Strategy)
BiCarta<BiomeType> myBiomeAlgorithm = (x, z, workerId) -> {
    if (Noise.get(x, z) > 0.5) {
        return BiomeTypes.FOREST;
    }
    return BiomeTypes.PLAINS;
};

// 2. Define the output buffer type
NParametrizedBufferType biomeBuffer = NBufferTypes.BIOME_ID;

// 3. Instantiate and configure the stage
NStage biomeStage = new NBiomeStage(
    "CoreBiomeGeneration",
    biomeBuffer,
    myBiomeAlgorithm
);

// 4. Add the stage to the generation pipeline
pipelineBuilder.addStage(biomeStage);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Invocation:** Never call the `run` method directly. This bypasses the pipeline executor, which is responsible for buffer management, dependency resolution, and thread scheduling. Bypassing the executor will lead to uninitialized state and likely a NullPointerException.
-   **Stateful BiCarta:** Do not inject a **BiCarta** that relies on mutable shared state. The stage may be executed by any worker thread at any time, and a stateful algorithm will cause severe and difficult-to-debug concurrency issues.
-   **Misconfigured Dependencies:** Do not attempt to modify the stage to depend on an input buffer. The design of **NBiomeStage** is explicitly for data creation, not transformation. If you need to modify existing biome data, a different type of stage should be used.

## Data Pipeline
As a source stage, **NBiomeStage** initiates a data flow. It transforms spatial coordinates into discrete biome data and writes that data into a shared buffer for consumption by subsequent stages.

> Flow:
> (x, z) Coordinates -> BiCarta Algorithm -> **NBiomeStage** -> NPixelBufferView Write -> NCountedPixelBuffer (Output)

