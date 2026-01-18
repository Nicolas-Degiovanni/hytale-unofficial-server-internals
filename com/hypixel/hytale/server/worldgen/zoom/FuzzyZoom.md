---
description: Architectural reference for FuzzyZoom
---

# FuzzyZoom

**Package:** com.hypixel.hytale.server.worldgen.zoom
**Type:** Transient

## Definition
```java
// Signature
public class FuzzyZoom {
```

## Architecture & Concepts

FuzzyZoom is a key component in the procedural world generation pipeline, responsible for transforming discrete, grid-based data into more organic, natural-looking patterns. It functions as a **Decorator** over an underlying ExactZoom instance, augmenting its behavior with controlled randomness.

The core purpose of this class is to mitigate the artificial, grid-like artifacts that can arise from purely deterministic sampling algorithms like ExactZoom. By applying a coordinate-based randomization function via the ICoordinateRandomizer interface, it perturbs the sampling coordinates before querying the underlying data source. This "fuzziness" is essential for creating smooth and believable transitions between world features, such as biome borders.

Architecturally, FuzzyZoom instances are intended to be chained together in a pipeline. A typical world generation pass will construct a series of zoom layers, where each layer refines or transforms the output of the previous one. FuzzyZoom is a critical step in this chain, breaking the rigid alignment of the procedural grid.

## Lifecycle & Ownership

-   **Creation:** FuzzyZoom instances are created on-demand by higher-level world generation orchestrators. They are not globally managed or injected as services. A new instance is typically constructed for each specific generation layer or pass that requires randomized sampling.
-   **Scope:** The lifetime of a FuzzyZoom object is short and strictly confined to the scope of the generation task it was created for. It persists only as long as it is needed to compute a specific layer of world data.
-   **Destruction:** The object is relinquished and becomes eligible for garbage collection as soon as the generation pass completes. Methods like generateUniqueZones follow a functional pattern, returning a *new* FuzzyZoom instance rather than mutating the existing one, reinforcing this transient lifecycle.

## Internal State & Concurrency

-   **State:** The internal state, consisting of the ICoordinateRandomizer and the wrapped ExactZoom, is established at construction and is not modified thereafter. The class is designed to be **effectively immutable**. Operations that transform the data layer produce new FuzzyZoom instances, preserving the integrity of the original.
-   **Thread Safety:** This class is **not inherently thread-safe**. Its safety is entirely dependent on the thread safety of its injected dependencies, specifically the ICoordinateRandomizer and the PixelProvider used by the composed ExactZoom. In typical engine usage, world generation for a given region is single-threaded, obviating concurrency concerns for a single instance.

    **Warning:** Do not share a single FuzzyZoom instance across multiple worker threads unless you can guarantee that all its underlying components are thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generate(x, y) | int | O(1) | Delegates directly to the underlying ExactZoom to sample a value at a precise coordinate, bypassing any fuzziness. |
| getX(seed, x, y) | double | O(1) | Computes a randomized X-coordinate based on the input location and seed. This is the core of the "fuzzy" behavior. |
| getY(seed, x, y) | double | O(1) | Computes a randomized Y-coordinate based on the input location and seed. |
| distance(x, y) | double | O(1) | Delegates to ExactZoom to calculate the distance to the nearest pixel boundary. |
| generateUniqueZones(...) | FuzzyZoom | O(N) | Creates a new data layer by placing unique zones based on candidate locations. Returns a **new** FuzzyZoom instance wrapping the result. |

## Integration Patterns

### Standard Usage

FuzzyZoom is used as a decorator in a chain of generation layers to introduce organic variation.

```java
// Context: Within a world generator for a specific layer (e.g., biomes)
ICoordinateRandomizer randomizer = getCoordinateRandomizer();
PixelProvider baseNoiseProvider = getBaseNoiseLayer();

// 1. Create a deterministic, grid-based zoom layer from the base noise
ExactZoom exactLayer = new ExactZoom(baseNoiseProvider, 2.0, 2.0, 0, 0);

// 2. Wrap the exact layer with a FuzzyZoom to break up grid artifacts
FuzzyZoom fuzzyLayer = new FuzzyZoom(randomizer, exactLayer);

// 3. Use the fuzzy layer to generate final values for world coordinates
int biomeId = fuzzyLayer.generate(worldX, worldY);
```

### Anti-Patterns (Do NOT do this)

-   **Bypassing Fuzziness:** Calling only the generate method on a FuzzyZoom instance completely ignores its primary purpose. If you do not need randomized sampling, use the ExactZoom class directly for better performance and clarity.
-   **Stateful Reuse:** Do not attempt to modify and reuse a FuzzyZoom instance across different generation passes. The design relies on creating new instances for new layers of data, as seen with the generateUniqueZones method. Treat instances as immutable snapshots of a generation stage.

## Data Pipeline

FuzzyZoom acts as a transformation stage in a larger data flow. It consumes a deterministic grid provider (via ExactZoom) and produces a randomized, non-uniform sampling interface.

> Flow:
> Base PixelProvider -> ExactZoom (Grid Sampling) -> **FuzzyZoom** (Coordinate Perturbation) -> Final Zone ID

