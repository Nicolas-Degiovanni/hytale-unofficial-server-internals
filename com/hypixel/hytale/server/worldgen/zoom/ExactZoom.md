---
description: Architectural reference for ExactZoom
---

# ExactZoom

**Package:** com.hypixel.hytale.server.worldgen.zoom
**Type:** Transient

## Definition
```java
// Signature
public class ExactZoom {
```

## Architecture & Concepts

The ExactZoom class is a fundamental component in the server-side procedural world generation pipeline. It does not represent a single algorithm but rather a stateful *view* or *transformation layer* on top of a discrete grid of data, represented by a PixelProvider. Its primary purpose is to bridge different scales of generation, allowing high-resolution feature placement to occur on low-resolution base maps.

Conceptually, ExactZoom acts as a coordinate space transformer and data sampler. It takes a base data grid (the source PixelProvider), applies a scale (zoom) and translation (offset), and presents a continuous coordinate system to subsequent generation stages. This allows algorithms to query for biome data or other world properties at any floating-point coordinate, which ExactZoom then resolves to the appropriate discrete pixel in the source grid.

Its most critical function is the procedural placement of unique, non-repeating world features, encapsulated by the Zone system. The class contains logic to identify valid candidate locations for these features and then "stamp" them onto the data grid. This process is functionally immutable; modifying the grid via feature placement results in the creation of a **new** ExactZoom instance containing the updated data, preserving the original as a base layer. This chaining of immutable transformations is central to the determinism and robustness of the world generator.

### Lifecycle & Ownership
- **Creation:** An ExactZoom instance is created by a higher-level world generation orchestrator. It is initialized with a PixelProvider from a preceding generation stage (e.g., a base continent map) and the specific transformation parameters (zoom, offset) for the current stage.
- **Scope:** The object is short-lived and scoped to a specific phase of world generation. It is created, used to perform sampling or feature placement, and then typically discarded. The result of its work is either a final data grid or a new ExactZoom instance that is passed to the next stage in the pipeline. It does not persist beyond the world generation process.
- **Destruction:** The object becomes eligible for garbage collection as soon as the generation pipeline advances and no longer holds a reference to it.

## Internal State & Concurrency
- **State:** The class is designed to be **effectively immutable**. All member fields are final. While the underlying PixelProvider could theoretically be mutable, the class's operational contract treats it as a read-only source for sampling operations. For mutation operations like generateUniqueZones, a deep copy of the source PixelProvider is created internally, ensuring the original instance and its data source remain unmodified. This prevents side effects between generation stages.
- **Thread Safety:** The class is **conditionally thread-safe**. Because its state is immutable, multiple threads can safely call read-only methods like generate or inBounds on a shared instance. However, the entire generation pipeline, especially state-transforming methods like generateUniqueZones, is designed for single-threaded execution to ensure deterministic outcomes.

**WARNING:** Do not share a single ExactZoom instance across multiple generation threads that perform write operations. The immutability-by-copying pattern is designed for a sequential, single-threaded pipeline. Concurrent calls to generateUniqueZones will not affect each other's source data but will result in a race condition over which result is used.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generate(double x, double y) | int | O(1) | Transforms the given coordinates to the source grid space and samples the corresponding pixel value. |
| distanceToNextPixel(double x, double y) | double | O(N) | Calculates the scaled distance to the nearest pixel with a different value. Computationally intensive. |
| generateUniqueZoneCandidates(...) | Zone.UniqueCandidate[] | O(W*H*R²) | Scans the entire source grid to find all valid positions for placing unique zones. Extremely high complexity. |
| generateUniqueZones(...) | ExactZoom | O(C*R²) | Attempts to place a set of unique zones onto a copy of the source grid and returns a **new** ExactZoom instance with the result. |
| inBounds(double x, double y) | boolean | O(1) | Checks if the given coordinates map to a valid location within the source grid. |

## Integration Patterns

### Standard Usage
ExactZoom is intended to be used as a stage in a functional pipeline. A base layer is created, and then successive calls to generateUniqueZones produce new, more detailed layers.

```java
// 1. Create a base layer from a previous generation step
PixelProvider baseProvider = ...;
ExactZoom baseZoom = new ExactZoom(baseProvider, 16.0, 16.0, 0, 0);

// 2. Find locations for and place unique features
Zone.UniqueCandidate[] candidates = baseZoom.generateUniqueZoneCandidates(entries, 16);
List<Zone.Unique> placedZones = new ArrayList<>();
ExactZoom finalZoom = baseZoom.generateUniqueZones(candidates, random, placedZones);

// 3. Use the final, modified layer for subsequent sampling
int biomeId = finalZoom.generate(1024.5, -512.3);
```

### Anti-Patterns (Do NOT do this)
- **Ignoring the Return Value:** The generateUniqueZones method does not modify the instance it is called on. It returns a new instance with the changes. Failing to capture this return value means the changes are lost.

  ```java
  // BAD: The changes are discarded, baseZoom is unmodified
  baseZoom.generateUniqueZones(candidates, random, placedZones);
  int biomeId = baseZoom.generate(x, y); // Samples from the original, unmodified data

  // GOOD: Capture the new instance
  ExactZoom updatedZoom = baseZoom.generateUniqueZones(candidates, random, placedZones);
  int biomeId = updatedZoom.generate(x, y); // Samples from the modified data
  ```

- **External Mutation of PixelProvider:** Modifying the source PixelProvider given to an ExactZoom constructor from another thread or process after creation will break the immutability contract and lead to non-deterministic and unpredictable generation results.

## Data Pipeline
ExactZoom is a clear example of a pipeline stage. Data flows through it, is transformed, and is then passed to the next stage.

> Flow:
> PixelProvider (From previous stage) -> **new ExactZoom(...)** -> generateUniqueZones(...) -> **New ExactZoom Instance** -> generate(...) -> Final Biome ID (Integer)

