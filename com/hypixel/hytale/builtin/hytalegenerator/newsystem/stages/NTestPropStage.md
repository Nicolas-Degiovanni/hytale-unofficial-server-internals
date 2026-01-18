---
description: Architectural reference for NTestPropStage
---

# NTestPropStage

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.stages
**Type:** Component / Transient

## Definition
```java
// Signature
public class NTestPropStage implements NStage {
```

## Architecture & Concepts
The NTestPropStage is a concrete implementation of the **NStage** interface, representing a single, atomic step within the world generation pipeline. Its specific function is *terrain decoration* by placing simple, vertical structures, referred to as "props", onto the generated landscape.

This class operates as a data transformer within the generator's execution flow. It reads from a source voxel buffer, identifies specific patterns in the voxel data, and writes new data to a destination buffer. Architecturally, it embodies the Strategy Pattern, where the world generator orchestrator invokes a series of these stages in a defined order to build the final world.

The core logic identifies a placement condition: an *anchor* material block sitting directly on top of a *floor* material block. Upon finding this pattern, it places a vertical column of a specified *prop* material. This mechanism is suitable for placing simple features like trees, pillars, or stalagmites.

## Lifecycle & Ownership
- **Creation:** NTestPropStage is instantiated and configured by a higher-level world generator orchestrator or a configuration system. The constructor requires all dependencies, including input/output buffer types and the specific materials for floor, anchor, and prop, ensuring each instance is fully configured upon creation.

- **Scope:** The lifetime of an NTestPropStage instance is ephemeral. It is designed to exist only for the duration of a single generation task, such as processing one world chunk. It holds no state that persists between separate `run` invocations.

- **Destruction:** The object is eligible for standard Java garbage collection once the orchestrator that created it has completed the `run` method and released its reference. It does not manage any unmanaged resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The internal state of NTestPropStage is **immutable** after construction. All configuration parameters, such as materials and buffer types, are stored in final fields. The `run` method is purely computational and does not modify the instance's internal state.

- **Thread Safety:** This class is **not thread-safe**. The `run` method operates on mutable buffer views provided by the NStage.Context. The world generation engine is responsible for guaranteeing thread safety.

    **WARNING:** The caller must ensure that no two NStage instances write to the same output buffer concurrently. An NTestPropStage instance should not be shared across multiple worker threads; each thread should be provided with its own unique instance.

## API Surface
The public contract is defined by the NStage interface and the public constructor.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NTestPropStage(input, output, floor, anchor, prop) | constructor | O(1) | Creates and configures a new stage instance. |
| run(NStage.Context context) | void | O(H) | Executes the prop placement logic. Complexity is linear to the vertical scan height. |
| getInputTypesAndBounds_bufferGrid() | Map | O(1) | Declares the required input buffer types and their spatial bounds. |
| getOutputTypes() | List | O(1) | Declares the output buffer types this stage will write to. |
| getName() | String | O(1) | Returns the unique identifier for this stage. |

## Integration Patterns

### Standard Usage
NTestPropStage is intended to be used as part of a larger sequence of generation stages. An orchestrator configures it with specific materials and adds it to the pipeline.

```java
// Conceptual example within a world generator orchestrator

// 1. Define materials and buffer types
NBufferType terrainBuffer = ...;
NBufferType decoratedBuffer = ...;
SolidMaterial grass = materialRegistry.get("grass");
SolidMaterial dirt = materialRegistry.get("dirt");
SolidMaterial woodLog = materialRegistry.get("wood_log");

// 2. Instantiate and configure the stage
NStage treePlacementStage = new NTestPropStage(
    terrainBuffer,
    decoratedBuffer,
    grass,  // Floor material
    dirt,   // Anchor material
    woodLog // Prop material
);

// 3. Add to the generation pipeline
pipeline.addStage(treePlacementStage);

// 4. The orchestrator will later invoke stage.run(context)
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not attempt to reuse a single NTestPropStage instance for different configurations. Its configuration is immutable. Create a new instance for each distinct prop type or material set.

- **Concurrent Execution:** Do not invoke the `run` method on an instance from multiple threads simultaneously, even with different contexts. This will lead to race conditions if the underlying buffers overlap. The execution engine must serialize access or provide thread-isolated data.

- **Incorrect Configuration:** Providing buffer types that do not match the expected data format (e.g., a buffer of integers when SolidMaterial is expected) will result in runtime ClassCastExceptions. The assertions in the constructor provide a basic safeguard.

## Data Pipeline
The flow of data through this component is linear and unidirectional. The stage acts as a transformation function on a stream of voxel data.

> Flow:
> NStage.Context -> Request Input NVoxelBufferView (Read) -> **NTestPropStage: Scan & Identify Pattern** -> Write to Output NVoxelBufferView (Write) -> Modified Voxel Data

