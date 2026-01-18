---
description: Architectural reference for ImageBuilder
---

# ImageBuilder

**Package:** com.hypixel.hytale.server.core.universe.world.worldmap.provider.chunk
**Type:** Transient

## Definition
```java
// Signature
class ImageBuilder {
```

## Architecture & Concepts
The ImageBuilder is a package-private, transient worker class responsible for the asynchronous generation of a single map tile image. It functions as a multi-stage data processing pipeline, transforming raw, three-dimensional world chunk data into a shaded, two-dimensional **MapImage**.

Architecturally, this class sits at the intersection of the world data layer (**ChunkStore**) and the map visualization system. Its primary design goals are performance and concurrency. It achieves this by aggressively parallelizing I/O-bound operations (fetching chunk data from storage) and deferring CPU-bound rendering calculations until all necessary data is available.

The entire process is orchestrated via a **CompletableFuture** chain, ensuring that data fetching from the primary chunk and its eight neighbors occurs concurrently. The final image generation is then executed as the terminal stage of this asynchronous pipeline. This design prevents the map generation system from blocking server threads on expensive disk or network I/O.

## Lifecycle & Ownership
- **Creation:** An ImageBuilder is instantiated exclusively through the static factory method **build**. Direct instantiation is an anti-pattern and will result in a non-functional object. The factory method immediately initiates the asynchronous data pipeline.
- **Scope:** The object's lifetime is ephemeral, strictly bound to the **CompletableFuture** chain returned by the **build** method. It exists solely to process one chunk and produce one image.
- **Destruction:** Once the **CompletableFuture** completes (either successfully or exceptionally) and the final **MapImage** is extracted, the ImageBuilder instance holds no further purpose and becomes eligible for garbage collection. There are no mechanisms for reset or reuse.

## Internal State & Concurrency
- **State:** The ImageBuilder is highly stateful and mutable. It contains numerous buffers (e.g., **heightSamples**, **tintSamples**, **blockSamples**) that are populated sequentially as the pipeline progresses. The state transitions from initial buffer allocation to data-filled, and finally to a completed state where the **image** field contains the final rendered data.

- **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. Its design relies on the "happens-before" guarantees of the **CompletableFuture** execution stages. All mutations to its internal state are performed by a single thread at each stage of the asynchronous chain. Sharing an ImageBuilder instance across multiple threads will result in race conditions and data corruption.

## API Surface
The primary public contract is the static **build** method. Instance methods are effectively package-private and should not be invoked directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(index, width, height, world) | CompletableFuture<ImageBuilder> | O(N*M) | **Static Factory.** Initiates the entire asynchronous image generation pipeline for a given chunk index. Returns a future that completes with the builder instance or null on failure. |
| getIndex() | long | O(1) | Returns the chunk index this builder is processing. |
| getImage() | MapImage | O(1) | Returns the final generated **MapImage**. WARNING: This is only valid after the future returned by **build** has successfully completed. |

## Integration Patterns

### Standard Usage
The only correct way to use this class is to invoke the static **build** method and process the result asynchronously.

```java
// How a developer should normally use this
CompletableFuture<ImageBuilder> futureBuilder = ImageBuilder.build(chunkIndex, 64, 64, world);

futureBuilder.thenAccept(builder -> {
    if (builder != null) {
        MapImage finalImage = builder.getImage();
        // Process the finalImage, e.g., send to client or cache.
    } else {
        // Handle cases where the chunk could not be loaded or processed.
    }
});
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ImageBuilder()`. This bypasses the entire asynchronous pipeline orchestration managed by the static **build** method.
- **Synchronous Blocking:** Do not call `.join()` or `.get()` on the returned future from a main server thread. This defeats the purpose of the asynchronous design and will cause server stalls.
- **State Tampering:** Do not attempt to access or modify the internal state of the ImageBuilder while the future is in-flight. The object's state is managed exclusively by the pipeline.

## Data Pipeline
The ImageBuilder executes a strict, ordered pipeline to transform a chunk index into a rendered image.

> Flow:
> Chunk Index -> **ImageBuilder.build()** -> [Concurrent Fetching] -> **ImageBuilder** (Internal State) -> [Rendering] -> MapImage

1.  **Initiation:** The static **build** method creates an ImageBuilder instance and initiates the pipeline.
2.  **Concurrent Data Fetching:**
    -   The primary chunk's data (**WorldChunk**, **FluidSection**) is fetched asynchronously from the **ChunkStore**.
    -   Simultaneously, heightmap data for all 8 neighboring chunks is fetched asynchronously from the **ChunkStore**. A **CompletableFuture.allOf** is used to await completion of all nine fetches.
3.  **Data Sampling & Rendering:** Once all data is loaded into memory, the **generateImageAsync** stage executes.
    -   It samples blocks, heights, tints, and fluids from the primary chunk into internal array buffers.
    -   It iterates through each pixel of the target image.
    -   For each pixel, it calculates a base color from the corresponding block data.
    -   It calculates a dynamic shading value using the height data of the pixel and its neighbors, creating a 3D lighting effect.
    -   It overlays fluid colors, factoring in depth and environmental tints.
    -   The final computed color is packed and written to the **MapImage** data buffer.
4.  **Completion:** The **CompletableFuture** completes, making the ImageBuilder instance, which now contains the final **MapImage**, available to the caller.

