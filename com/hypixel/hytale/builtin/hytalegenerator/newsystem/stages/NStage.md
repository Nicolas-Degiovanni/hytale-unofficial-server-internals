---
description: Architectural reference for the NStage interface, the fundamental contract for world generation steps.
---

# NStage

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.stages
**Type:** Contract Interface

## Definition
```java
// Signature
public interface NStage {
   void run(@Nonnull NStage.Context var1);

   @Nonnull
   Map<NBufferType, Bounds3i> getInputTypesAndBounds_bufferGrid();

   @Nonnull
   List<NBufferType> getOutputTypes();

   @Nonnull
   String getName();

   // ... nested Context class
}
```

## Architecture & Concepts
The NStage interface is the foundational contract for all procedural world generation operations within Hytale's new generator system. It represents a single, atomic, and potentially parallelizable step in the complex pipeline of creating a game world. Each implementation of NStage, referred to as a "stage", is a self-contained algorithm that reads from a set of input data buffers, performs a transformation, and writes to a set of output data buffers.

This design decouples the generation logic (e.g., "CarveCaves", "GenerateOres", "ApplyBiomes") from the execution engine. The engine, or scheduler, is responsible for building a dependency graph of stages based on their declared inputs and outputs. It then executes these stages, often in parallel across multiple worker threads, to efficiently generate world chunks.

The core architectural principles are:
- **Modularity:** Each stage is a distinct unit of work, promoting code reuse and simplifying the overall generation logic.
- **Parallelism:** The contract is explicitly designed for concurrency. Stages are expected to be stateless and re-entrant, allowing the scheduler to run many instances on different data sets simultaneously.
- **Data-Driven:** Stages declare their data dependencies upfront. This allows the system to manage memory and schedule tasks efficiently, ensuring a stage only runs when its required input data is available.

## Lifecycle & Ownership
- **Creation:** Implementations of NStage are instantiated once during the initialization of the world generation system. They are typically registered with a central `StageRegistry` or scheduler.
- **Scope:** An NStage implementation is a long-lived, application-scoped object. It persists for the entire lifetime of the world generator, acting as a reusable template for a specific generation task.
- **Destruction:** NStage objects are garbage collected when the world generation system is shut down or reconfigured. The `Context` object passed to the `run` method is, by contrast, extremely short-lived and scoped only to a single execution of that task.

## Internal State & Concurrency
- **State:** The NStage interface contract **mandates** that implementations be stateless. All necessary data for an operation is provided via the `NStage.Context` parameter in the `run` method. Storing mutable instance variables on an NStage implementation is a severe violation of the design and will lead to catastrophic race conditions and non-deterministic world generation.
- **Thread Safety:** Implementations **must be** thread-safe and re-entrant. The `run` method will be invoked concurrently by multiple worker threads on different world regions. All operations must be confined to the data buffers provided in the `Context`. The provided `WorkerIndexer.Id` can be used for accessing thread-local resources or seeding thread-specific random number generators.

## API Surface
The public contract of NStage is focused on metadata declaration and task execution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| run(Context) | void | O(N) | The primary execution method. Performs the generation logic on the data buffers supplied in the Context. N is the volume of the bounds. |
| getInputTypesAndBounds_bufferGrid() | Map | O(1) | Declares the data dependencies. Returns a map of required buffer types and the spatial bounds from which data is needed. |
| getOutputTypes() | List | O(1) | Declares the data this stage produces. Returns a list of buffer types that this stage will write to. |
| getName() | String | O(1) | Provides a human-readable name for the stage, used for debugging, logging, and profiling. |

## Integration Patterns

### Standard Usage
A developer does not call NStage methods directly. Instead, they create a class that implements the NStage interface and register it with the world generation pipeline. The system's scheduler then invokes the stage at the appropriate time.

```java
// 1. Implement the interface for a specific task
public class OrePlacementStage implements NStage {
    @Override
    public void run(@Nonnull NStage.Context context) {
        // Access input buffers for stone type, heightmaps, etc.
        NBufferBundle.Access.View blockBuffer = context.bufferAccess.get(NBufferTypes.BLOCKS);
        
        // ... algorithm to place ores ...
        
        // Write results into the same buffer
        blockBuffer.set(x, y, z, ORE_BLOCK_ID);
    }

    @Override
    public Map<NBufferType, Bounds3i> getInputTypesAndBounds_bufferGrid() {
        // Declare that we need read-write access to the main block buffer
        return Map.of(NBufferTypes.BLOCKS, new Bounds3i(0, 0, 0, 16, 256, 16));
    }

    // ... other method implementations ...
}

// 2. Register the stage with the generator (conceptual)
worldGenerator.registerStage(new OrePlacementStage());
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Storing any mutable state as a member variable of an NStage implementation is the most critical anti-pattern. This breaks thread safety and will cause unpredictable generation bugs that are difficult to reproduce.
- **Direct Invocation:** Manually creating a `Context` and calling `run` bypasses the dependency scheduler and buffer management system. This will fail, as the required data buffers will not be present or correctly configured.
- **Cross-Thread Communication:** A stage must not communicate with other threads or other running stages. It must operate solely on the data provided in its `Context`.

## Data Pipeline
NStage is a component in a larger data processing pipeline. The scheduler orchestrates the flow of data between stages.

> Flow:
> Previous Stage Output Buffers -> Scheduler resolves dependencies -> **NStage.Context** (provides buffer views) -> **NStage.run()** modifies buffers -> Output is available as input for subsequent stages

