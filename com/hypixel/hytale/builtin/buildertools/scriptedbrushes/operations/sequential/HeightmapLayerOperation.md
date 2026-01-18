---
description: Architectural reference for HeightmapLayerOperation
---

# HeightmapLayerOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient Component

## Definition
```java
// Signature
public class HeightmapLayerOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The HeightmapLayerOperation is a specialized component within the Scripted Brush framework, designed to execute a single, stateful step in a sequence of world modifications. Its primary architectural role is to translate vertical position into material composition, effectively "painting" layers onto terrain based on depth from the surface.

This operation is entirely data-driven, configured through a list of LayerEntryCodec objects. Each entry defines a thickness and a material. The operation processes blocks from the top down, consuming depth from its layer list until it finds the appropriate material for a given block's depth relative to the highest point in its column.

A key feature is its dual-mode material resolution. It can resolve block types from either a static asset name (e.g., "hytale:stone") or a dynamic argument supplied by the player's active BuilderTool. This allows for the creation of highly generic and reusable brushes that can be customized in-game by the player. It acts as a bridge between a declarative brush configuration and the imperative logic of world modification.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly via a constructor. They are deserialized and instantiated by the static HeightmapLayerOperation.CODEC when the engine loads a BuilderTool asset configuration. This process populates the internal layerArgs list from the asset data.

- **Scope:** The object's lifetime is bound to the execution of a single brush application. It is held by the parent BrushConfig object and is used for every block processed within the brush's volume of effect. It does not persist between distinct uses of the tool.

- **Destruction:** The object is eligible for garbage collection once the brush application is complete and the parent BrushConfig is discarded. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
- **State:** The primary state is the `layerArgs` list, which contains the configured layers. This state is populated once during creation by the codec and is treated as immutable for the remainder of the object's lifecycle. The operation is therefore stateful but its state is static post-initialization.

- **Thread Safety:** **Not Thread-Safe.** This component is designed to be executed exclusively on the main server thread as part of the game loop. It modifies a BrushConfigEditStore, which is a non-thread-safe transactional buffer for block changes. All calls must be synchronized by the parent brush execution system.

## API Surface
The public contract is minimal, focused on its role as a step in a larger sequence.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBlocks(...) | boolean | O(N) | Core logic. Calculates block depth and applies the corresponding layer material to the BrushConfigEditStore. N is the number of configured layers. |
| modifyBrushConfig(...) | void | O(1) | Hook for modifying the parent brush configuration. This operation provides an empty implementation as it does not alter the brush itself. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in Java code. It is configured declaratively within a BuilderTool's asset definition file. The engine's codec system handles its instantiation and integration into the brush lifecycle.

A typical configuration would resemble the following conceptual structure:

```yaml
# Inside a BuilderTool asset file (e.g., .hbt)
name: "Terraforming Brush"
operations:
  - type: "HeightmapLayer"
    Layers:
      - { depth: 1, material: "hytale:grass" }
      - { depth: 3, material: "hytale:dirt" }
      - { depth: 0, material: "hytale:stone" } # 0 means infinite depth
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new HeightmapLayerOperation()`. This creates an object with an empty layer list, rendering it useless. Always define it within an asset file to be loaded by the engine's codec.

- **Manual Invocation:** Do not call the `modifyBlocks` method from custom game logic. This bypasses the necessary context, such as the BrushConfigEditStore and ComponentAccessor, provided by the scripted brush system, leading to unpredictable behavior and world corruption.

## Data Pipeline
The flow of data for this operation is linear and unidirectional, transforming a high-level configuration into low-level world edits.

> Flow:
> BuilderTool Asset File -> Engine Codec Deserialization -> **HeightmapLayerOperation Instance** -> Brush Execution System -> modifyBlocks(x,y,z) -> BrushConfigEditStore -> WorldChunk Update

