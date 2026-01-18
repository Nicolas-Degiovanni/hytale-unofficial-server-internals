---
description: Architectural reference for LayeredCarta
---

# LayeredCarta

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.cartas
**Type:** Transient

## Definition
```java
// Signature
public class LayeredCarta<R> extends TriCarta<R> {
```

## Architecture & Concepts
The LayeredCarta is a composite data structure that implements the TriCarta functional interface. It acts as a container for an ordered sequence of other TriCarta instances, referred to as *layers*. Its primary role within the world generation framework is to combine multiple data sources into a single, coherent result based on a priority system.

When queried for a value at a specific 3D coordinate, the LayeredCarta iterates through its layers in the order they were added. It returns the value from the *last* layer that provides a non-null result. If all layers return null for the given coordinate, a pre-configured default value is returned.

This layering mechanism is a foundational pattern for procedural generation. It allows developers to build complex features by composing simpler, single-purpose data maps. For example, a base terrain layer can be overlaid with a biome-specific block layer, which is then overlaid with a structure-specific layer, with each subsequent layer having the ability to override the ones beneath it.

## Lifecycle & Ownership
- **Creation:** A LayeredCarta is instantiated directly by a higher-level component responsible for orchestrating a specific world generation phase. It is not managed by a dependency injection framework or a central registry.
- **Scope:** The lifetime of a LayeredCarta is typically bound to a single, complete world generation task. It is configured with all its necessary layers at the beginning of the task and then used for the duration of that task.
- **Destruction:** The object is eligible for garbage collection as soon as the generation task completes and all references to it are released. There is no explicit cleanup or disposal method.

## Internal State & Concurrency
- **State:** The internal state is **mutable**. The LayeredCarta maintains an ordered list of its child TriCarta layers and an aggregated list of all possible return values. This state is modified exclusively through the addLayer method.

- **Thread Safety:** This class is **conditionally thread-safe**.
    - **WARNING:** The class is **NOT** safe for concurrent modification. The addLayer method modifies internal collections without any synchronization. Calling addLayer from multiple threads, or while another thread is calling apply, will lead to race conditions and undefined behavior.
    - The apply method is safe for concurrent reads, provided that no modifications (calls to addLayer) are occurring simultaneously. The intended pattern is to fully configure the LayeredCarta instance on a single thread and then share it for read-only access across multiple worker threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| LayeredCarta(defaultValue) | constructor | O(1) | Creates a new instance with a mandatory default value. |
| apply(x, y, z, id) | R | O(N) | Queries all N layers in order and returns the last non-null result, or the default value. |
| addLayer(layer) | LayeredCarta<R> | O(M) | Adds a new TriCarta layer. Complexity is dominated by adding the layer's M possible values to the internal cache. |
| allPossibleValues() | List<R> | O(1) | Returns an unmodifiable view of all possible values from the default value and all added layers. |

## Integration Patterns

### Standard Usage
The standard pattern is to instantiate, configure, and then use the LayeredCarta. The configuration phase must be completed before the object is passed to any concurrent execution context, such as world generation workers.

```java
// How a developer should normally use this
// 1. Create with a default value (e.g., AIR block)
LayeredCarta<Block> carta = new LayeredCarta<>(Blocks.AIR);

// 2. Configure by adding layers in order of increasing priority
TriCarta<Block> baseTerrain = createBaseTerrainCarta();
TriCarta<Block> oreVeins = createOreVeinCarta();
carta.addLayer(baseTerrain);
carta.addLayer(oreVeins); // Ore veins will override base terrain

// 3. Use in a read-only fashion, potentially across multiple threads
Block result = carta.apply(10, 50, 20, workerId);
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never call addLayer after the LayeredCarta has been shared with worker threads or while the apply method could be executing on another thread. This is the most critical anti-pattern and will corrupt the internal state.
- **Relying on Layer Order:** The order of addLayer calls is significant. Layers added later have higher priority. Reversing the order will produce different world generation results.
- **Null Default Value:** The constructor will throw a NullPointerException if a null default value is provided. The system relies on having a non-null fallback.

## Data Pipeline
The LayeredCarta functions as a data aggregator or multiplexer within the world generation pipeline. It takes a coordinate as input and transforms it into a final data value by consulting a series of prioritized data sources.

> Flow:
> 3D Coordinate -> **LayeredCarta**.apply() -> [Query Layer 1, Query Layer 2, ..., Query Layer N] -> Final Value (type R) -> World Chunk Data

