---
description: Architectural reference for StripedMaterialProvider
---

# StripedMaterialProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders
**Type:** Transient

## Definition
```java
// Signature
public class StripedMaterialProvider<V> extends MaterialProvider<V> {
```

## Architecture & Concepts
The StripedMaterialProvider is a compositional filter within the procedural world generation system. It functions as a Decorator for another MaterialProvider, constraining its material placement to a set of predefined horizontal bands, or "stripes", along the world's Y-axis.

Its primary role is to enable layered geological or environmental features. For example, a designer can use this provider to ensure that a specific ore, provided by a wrapped NoiseMaterialProvider, only appears between Y-levels 10 and 20. By composing multiple providers, complex strata can be built.

This class acts as a spatial gate. If a request for a voxel type falls within one of its configured Y-axis stripes, it delegates the request to its underlying provider. If the request is outside all defined stripes, it returns null, effectively vetoing any material placement at that location.

### Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level world generation orchestrator, typically when parsing a world generation configuration file. It is not intended for direct instantiation during the game loop.
- **Scope:** The object's lifetime is scoped to the world generation task for which it was configured. It is a short-lived object used during the chunk generation phase.
- **Destruction:** The object holds no native resources and requires no explicit cleanup. It becomes eligible for garbage collection as soon as the generation pass that created it is complete.

## Internal State & Concurrency
- **State:** The internal state consists of a reference to the decorated MaterialProvider and an array of Stripe objects. This state is established during construction and is **immutable** thereafter. The constructor performs a defensive copy of the incoming list of stripes into a private array.
- **Thread Safety:** This class is inherently **thread-safe**. Its immutable state guarantees that concurrent calls to getVoxelTypeAt from multiple world generation threads will not result in race conditions or data corruption. Thread safety of the overall operation depends on the thread safety of the wrapped MaterialProvider.

## API Surface
The public contract is minimal, adhering to the MaterialProvider interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getVoxelTypeAt(Context context) | V | O(N) | Evaluates the context's Y-position against N configured stripes. If a match is found, it delegates to the wrapped provider. Returns null if no stripe contains the position. |

**WARNING:** The time complexity is linear with respect to the number of stripes. Configurations with an excessive number of stripes may introduce performance bottlenecks in chunk generation.

## Integration Patterns

### Standard Usage
The provider is used to wrap and filter another provider. The standard pattern involves defining the stripes and the base material source, then composing them.

```java
// 1. Define the base material provider (e.g., always provides STONE)
MaterialProvider<VoxelType> stoneProvider = new ConstantMaterialProvider<>(STONE);

// 2. Define the horizontal stripes where the material can appear
List<StripedMaterialProvider.Stripe> stripes = List.of(
    new StripedMaterialProvider.Stripe(/*topY*/ 20, /*bottomY*/ 10),
    new StripedMaterialProvider.Stripe(/*topY*/ 5, /*bottomY*/ 0)
);

// 3. Create the StripedMaterialProvider, wrapping the base provider
MaterialProvider<VoxelType> stripedStoneProvider = new StripedMaterialProvider<>(stoneProvider, stripes);

// 4. During world generation, this provider is queried
//    - A call at Y=15 will return STONE.
//    - A call at Y=8 will return null.
VoxelType result = stripedStoneProvider.getVoxelTypeAt(generationContext);
```

### Anti-Patterns (Do NOT do this)
- **Overlapping Stripes:** The implementation iterates through the stripes and returns on the first match. If stripes overlap, only the one that appears earliest in the initial list will ever be effective for the overlapping region. This can lead to non-obvious generation bugs. Ensure stripe definitions are mutually exclusive.
- **Empty Stripe List:** Constructing this provider with an empty list of stripes is a valid but often incorrect configuration. It will cause getVoxelTypeAt to *always* return null, effectively silencing the wrapped provider entirely.
- **Large Stripe Count:** Avoid using this provider with thousands of stripes for performance reasons. If fine-grained vertical control is needed, consider a more specialized provider, such as one driven by a 1D noise function.

## Data Pipeline
The StripedMaterialProvider acts as a conditional passthrough in the material generation pipeline.

> Flow:
> World Generation Engine -> MaterialProvider.Context -> **StripedMaterialProvider** (Y-Axis Check) -> [Delegate] MaterialProvider -> Voxel Type (or null) -> Chunk Voxel Buffer

