---
description: Architectural reference for LayerOperation
---

# LayerOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient Operation

## Definition
```java
// Signature
public class LayerOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The LayerOperation is a specialized, data-driven component within the Scripted Brushes framework. It is not a standalone service but rather a single, configurable step within a larger sequence of world modification operations. Its primary architectural role is to implement a procedural layering algorithm, commonly used for generating natural-looking terrain surfaces.

This operation is designed to be defined declaratively within asset configuration files. The engine's `CODEC` system deserializes these configurations into LayerOperation instances at runtime. When a player uses a builder tool, the brush executor iterates through the affected volume, invoking this operation for each block column. The operation then calculates the vertical distance from the first air block it encounters and replaces the original block with a material defined in its configuration, based on this calculated depth.

This design decouples the complex logic of terrain generation from the core brush system, allowing game designers to create sophisticated layering behaviors without writing new Java code.

### Lifecycle & Ownership
- **Creation:** A LayerOperation is never instantiated directly using its constructor. It is exclusively created by the static `CODEC` field, which deserializes a brush's configuration from an asset file. This process occurs when a builder tool's brush definition is loaded by the server.
- **Scope:** The object's lifetime is extremely short and is scoped to a single brush application. When a player performs an action, a new set of operations is prepared, used to modify the world, and then immediately becomes eligible for garbage collection.
- **Destruction:** The instance is destroyed by the Java garbage collector once the parent brush execution completes. It holds no persistent state or native resources that require explicit cleanup.

## Internal State & Concurrency
- **State:** The primary state is the `layerArgs` list, which contains the depth and material information for each layer. This state is populated once during deserialization and is treated as immutable for the lifetime of the object. The class is therefore stateful but its configuration is fixed post-creation.
- **Thread Safety:** This class is **not thread-safe** and must not be accessed from multiple threads. It is designed to be executed exclusively within the server's main world-update thread. The `modifyBlocks` method mutates a `BrushConfigEditStore`, which is a non-thread-safe buffer for a single world modification transaction. Any concurrent access will result in severe world corruption and server instability.

## API Surface
The public API is minimal, as the class is primarily controlled by the brush execution engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBlocks(...) | boolean | O(D) | Core execution method. Called per-block. Scans vertically up to the max configured depth (D) to find air, then applies a block change. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class directly in Java code. Instead, they define its behavior declaratively in a builder tool's asset file. The engine handles the instantiation and execution.

A conceptual configuration might look like this:

```json
{
  "name": "terrain_layer_brush",
  "operations": [
    {
      "type": "Layer",
      "Layers": [
        { "depth": 1, "material": "hytale:grass" },
        { "depth": 3, "material": "hytale:dirt" },
        { "depth": 0, "material": "hytale:stone" } // 0 depth means infinite
      ]
    }
  ]
}
```

The brush system would then parse this and invoke the operation internally.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new LayerOperation()`. The object will be unconfigured and will fail at runtime. It must be created via the `CODEC` system from a valid configuration.
- **State Mutation:** Do not attempt to get and modify the internal `layerArgs` list after the object has been created. This is not a supported use case and will lead to unpredictable behavior.
- **Asynchronous Execution:** Do not pass an instance of this class to another thread for processing. All interactions must occur on the main server thread that owns the target `BrushConfigEditStore`.

## Data Pipeline
The LayerOperation functions as a processor in two key data flows: the world modification pipeline and the material resolution pipeline.

**World Modification Flow**
> Player Input (Use Tool) -> Brush Executor -> **LayerOperation.modifyBlocks** -> BrushConfigEditStore (Block Buffer) -> World Storage

**Material Resolution Flow**
The `resolveBlockId` method dynamically determines which block to place, pulling data from either static assets or the player's current tool.

> Flow:
> 1. **LayerOperation.modifyBlocks** triggers a material lookup.
> 2. The operation checks if the layer entry requires a dynamic tool argument.
> 3. **If Yes:** Player Entity -> Active Inventory Slot -> ItemStack -> BuilderTool Asset -> Tool Arguments -> **BlockPattern** -> Resolved Block ID
> 4. **If No:** Layer Configuration -> Material String (e.g., "hytale:stone") -> BlockType Asset Map -> Resolved Block ID

