---
description: Architectural reference for ReferenceBundle
---

# ReferenceBundle

**Package:** com.hypixel.hytale.builtin.hytalegenerator.referencebundle
**Type:** Transient

## Definition
```java
// Signature
public class ReferenceBundle {
```

## Architecture & Concepts
The ReferenceBundle is a specialized, transient data container designed to aggregate and transport multiple, distinct data layers during a complex, multi-stage process. Its primary application is within the world generation pipeline, where it serves as a "context object" or "scratchpad" passed between different generator stages.

Architecturally, it functions as a type-safe, string-keyed map. Each entry, or "layer," represents a specific facet of the generated world, such as a heightmap, biome data, or cave system information. The system uses a dual-map implementation internally:
1.  **Data Map:** A `HashMap<String, Reference>` for direct, high-performance access to the raw data objects.
2.  **Type Map:** A parallel `HashMap<String, String>` that stores the fully qualified class name of each corresponding data object.

This dual-map strategy provides a critical runtime safety feature. When a consumer requests a layer with a specific type, the ReferenceBundle first validates the stored type name against the requested type name before returning the object. This prevents `ClassCastException` errors and ensures that generator stages do not receive data in an unexpected format, returning null instead.

## Lifecycle & Ownership
-   **Creation:** A ReferenceBundle is instantiated via its public constructor (`new ReferenceBundle()`) at the beginning of a discrete, stateful operation. Within the Hytale Generator, this is typically done by an orchestrator or task runner responsible for a single unit of work, such as generating one world chunk.
-   **Scope:** The object's lifetime is strictly bound to the scope of the single generation task it was created for. It is populated incrementally as various generator stages execute and is considered complete only when the parent task finishes.
-   **Destruction:** The ReferenceBundle holds no persistent state and requires no explicit cleanup. It becomes eligible for garbage collection as soon as the orchestrating task completes and all references to the bundle are released.

## Internal State & Concurrency
-   **State:** The ReferenceBundle is fundamentally **mutable**. Its primary purpose is to accumulate state from multiple sources over its short lifespan. The internal maps are modified with every call to the `put` method.
-   **Thread Safety:** This class is **not thread-safe**. The underlying `HashMap` collections are unsynchronized. Concurrent access from multiple threads, especially simultaneous `put` operations, will result in race conditions, lost updates, and potential corruption of the bundle's internal state.

    **WARNING:** All interactions with a single ReferenceBundle instance must be confined to a single thread or be protected by external synchronization mechanisms. Do not share a ReferenceBundle across parallel generation tasks without explicit locking.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| put(name, reference, type) | void | O(1) | Adds or overwrites a named data layer. Stores both the reference and its class name for type validation. |
| getLayerWithName(name) | Reference | O(1) | Retrieves a data layer by name without type checking. Returns null if the layer does not exist. |
| getLayerWithName(name, type) | T extends Reference | O(1) | Retrieves a data layer by name, returning it only if its stored type matches the requested type. This is the preferred method for safe data access. |

## Integration Patterns

### Standard Usage
The canonical use case involves an orchestrator creating a bundle and passing it sequentially through a chain of processors. Each processor can add new layers or read existing layers to inform its logic.

```java
// 1. Orchestrator creates a new, empty bundle for a generation task.
ReferenceBundle bundle = new ReferenceBundle();

// 2. The first stage (e.g., terrain height) adds its data.
HeightmapGenerator heightmapGen = new HeightmapGenerator();
HeightmapReference heightmap = heightmapGen.createHeightmap();
bundle.put("terrain_height", heightmap, HeightmapReference.class);

// 3. A subsequent stage consumes the heightmap data to place biomes.
BiomeGenerator biomeGen = new BiomeGenerator();
HeightmapReference inputHeight = bundle.getLayerWithName("terrain_height", HeightmapReference.class);
if (inputHeight != null) {
    BiomeReference biomes = biomeGen.placeBiomes(inputHeight);
    bundle.put("biome_data", biomes, BiomeReference.class);
}
```

### Anti-Patterns (Do NOT do this)
-   **Bundle Reuse:** Do not reuse a ReferenceBundle instance across multiple, independent generation tasks (e.g., for two different chunks). This will cause data from one task to leak into the other, resulting in severe and difficult-to-diagnose generation bugs. Always create a new bundle for each new unit of work.
-   **Concurrent Modification:** Do not pass a ReferenceBundle to multiple threads that can write to it simultaneously without external locks. This will corrupt the bundle.
-   **Unsafe Getters:** Avoid using the non-typed `getLayerWithName(String name)` followed by a manual cast. This defeats the built-in type safety mechanism. The typed `getLayerWithName(String name, Class<T> type)` is strongly preferred.

## Data Pipeline
The ReferenceBundle does not process data itself; rather, it *is* the data carrier for the pipeline. It acts as a shared context that flows through and is mutated by each stage.

> Flow:
> Generation Task Start -> **new ReferenceBundle()** -> Stage 1 (adds Layer A) -> Stage 2 (reads Layer A, adds Layer B) -> Stage N (reads Layers A, B, ...) -> Generation Task End -> Final Data Product -> **ReferenceBundle is discarded**

