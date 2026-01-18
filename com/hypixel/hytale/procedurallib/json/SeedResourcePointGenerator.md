---
description: Architectural reference for SeedResourcePointGenerator
---

# SeedResourcePointGenerator

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class SeedResourcePointGenerator extends PointGenerator {
```

## Architecture & Concepts
The SeedResourcePointGenerator is a specialized implementation of the abstract PointGenerator. Its primary architectural role is to act as an **Adapter** between a data-centric SeedResource object and the logic-centric procedural generation pipeline.

In essence, it allows the procedural engine to treat a set of pre-defined points, loaded from an external resource like a JSON file, as if they were generated algorithmically in real-time. This is a critical component for enabling data-driven design in world generation, where artists or designers can define specific point clouds or features that are then seamlessly integrated into the larger procedural system.

The class achieves this by implementing the **Delegation Pattern**. It encapsulates a SeedResource instance and forwards all core data retrieval calls—such as requests for bounds or data buffers—directly to this encapsulated object. This design decouples the procedural algorithms from the source of the data, whether it be a file or a mathematical function.

## Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level factory or deserializer that processes world generation asset files. A SeedResource is loaded and parsed first, and then passed into this generator's constructor to wrap it.
- **Scope:** The lifetime of a SeedResourcePointGenerator is typically short and bound to a specific, self-contained procedural generation task. It is not a long-lived service.
- **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as the procedural generation pass that created it completes and releases its reference. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** This class is effectively immutable. Its internal state consists of a single *final* reference to a SeedResource. It performs no writes or modifications to its own state or the state of the encapsulated resource. It acts as a read-only view.
- **Thread Safety:** The thread safety of this class is entirely inherited from the injected SeedResource instance. As this class introduces no mutable state of its own, it is safe for concurrent use **if and only if** the underlying SeedResource is also thread-safe or is not mutated by other threads during the generation process.

## API Surface
The public contract is defined by the parent PointGenerator. The following protected methods constitute its core implementation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| localBounds2d() | ResultBuffer.Bounds2d | O(1) | Delegates to the encapsulated SeedResource to retrieve the 2D bounding box of the point data. |
| localBuffer2d() | ResultBuffer.ResultBuffer2d | O(1) | Delegates to the encapsulated SeedResource to retrieve the 2D point data buffer. |
| localBuffer3d() | ResultBuffer.ResultBuffer3d | O(1) | Delegates to the encapsulated SeedResource to retrieve the 3D point data buffer. |

## Integration Patterns

### Standard Usage
The intended use is to wrap a loaded SeedResource to make it compatible with systems that operate on the PointGenerator abstraction.

```java
// 1. A SeedResource is loaded from an asset file by a manager
SeedResource loadedResource = assetManager.load("data/worldgen/ore_vein_points.json");

// 2. The resource is adapted into a PointGenerator
PointGenerator fileBasedGenerator = new SeedResourcePointGenerator(
    worldSeedOffset,
    new CellularDistanceFunction(),
    new OrePointEvaluator(),
    loadedResource
);

// 3. The procedural engine consumes the generator transparently
proceduralEngine.addFeature(fileBasedGenerator);
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not modify the underlying SeedResource object after it has been passed to the generator's constructor, especially while a generation task is in progress. This class assumes the resource data is stable for its entire lifetime.
- **Data Mismatch:** Do not provide a SeedResource containing only 2D data to a system that will subsequently call localBuffer3d. This will result in runtime exceptions or undefined behavior. The provided resource must match the data dimensionality expected by the consumer.

## Data Pipeline
This component acts as a bridge, translating file-based data into a format consumable by the procedural logic pipeline.

> Flow:
> JSON Asset File -> Asset Deserializer -> **SeedResource** (In-Memory Data) -> **SeedResourcePointGenerator** (Adapter) -> Procedural Engine Consumer

