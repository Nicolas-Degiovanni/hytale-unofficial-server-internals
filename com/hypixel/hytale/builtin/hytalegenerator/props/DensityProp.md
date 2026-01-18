---
description: Architectural reference for DensityProp
---

# DensityProp

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props
**Type:** Transient Component

## Definition
```java
// Signature
public class DensityProp extends Prop {
```

## Architecture & Concepts
The DensityProp is a specialized world generation component responsible for placing volumetric, non-uniform features defined by a mathematical density function. It serves as a high-level orchestrator that combines multiple, more granular components to achieve a final result, such as ore veins, patches of dirt within stone, or complex organic structures.

Unlike simpler props that might place a single block or a fixed schematic, DensityProp operates in a multi-step process to generate emergent shapes within a defined volume. It does not contain the logic for the shape, material selection, or placement criteria itself. Instead, it delegates these responsibilities to injected strategy objects:

*   **Scanner:** Determines the root locations where a density feature should be attempted.
*   **Density:** A function that returns a scalar value for any point in 3D space. Positive values are considered "solid," and non-positive values are "air," effectively defining the feature's shape.
*   **MaterialProvider:** A complex rule engine that selects the appropriate block Material for a given solid voxel based on contextual data, such as its depth from a surface.
*   **Pattern & BlockMask:** Provide constraints on where the feature can be placed and what existing blocks it is allowed to overwrite.

The core design follows the two-phase **Scan-then-Place** pattern common throughout the world generator. The `scan` method first identifies all valid anchor points. Later, in a separate stage, the `place` method is called for each successful scan result to perform the computationally expensive task of sampling the density field and modifying the world's voxel data.

## Lifecycle & Ownership
-   **Creation:** A DensityProp is not instantiated directly by the engine's core loop. It is constructed as part of a larger world generation "recipe," typically defined within a biome or zone configuration. It is created with all its behavioral dependencies (Density, Scanner, etc.) supplied via its constructor.
-   **Scope:** The object's lifetime is tied to a specific world generation task. It is created, used by worker threads to process a set of world chunks, and is then eligible for garbage collection. It does not persist between sessions or across large-scale generation regions.
-   **Destruction:** Managed entirely by the Java garbage collector. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** The DensityProp is stateful but effectively **immutable after construction**. All dependency objects and configuration parameters are stored in final fields. The constructor performs significant pre-computation to determine the data dependencies (`ContextDependency`) and spatial footprint (`writeBounds_voxelGrid`) of the prop. This information is cached for its lifetime to inform the world generation scheduler.
-   **Thread Safety:** This class is designed to be used in a multi-threaded environment but is **not inherently thread-safe** in a general-purpose sense. An instance of DensityProp is intended to be used by a single worker thread at a time. The `scan` and `place` methods are not re-entrant and create transient state internally (e.g., the `densitySpace` buffer).

    **WARNING:** The design relies on the calling system, typically a staged conveyor or scheduler, to ensure that a single DensityProp instance is not accessed concurrently. The primary shared resource, the `VoxelSpace` (materialSpace), must be partitioned or employ its own locking mechanism to handle writes from multiple workers operating on different props or positions.

## API Surface
The public API exposes the two main phases of the generation process and provides metadata to the scheduler.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| scan(position, materialSpace, id) | PositionListScanResult | O(N) | Delegates to the injected Scanner to find valid anchor points for placement. N is the complexity of the scanner and pattern. |
| place(context) | void | O(R³) | Executes the primary generation logic. Samples the density field and places materials within its configured range. R is the size of the `range` vector. |
| getContextDependency() | ContextDependency | O(1) | Returns pre-computed data describing the read/write regions required by this prop. Critical for the generation scheduler. |
| getWriteBounds() | Bounds3i | O(1) | Returns the pre-computed maximum bounding box this prop can modify. |

## Integration Patterns

### Standard Usage
A DensityProp is not invoked directly. It is configured and submitted to the world generation framework, which manages the scan and place lifecycle across multiple worker threads.

```java
// Conceptual example within a generator configuration
// 1. Define the components
Density myDensity = new PerlinDensity(...);
MaterialProvider myMaterials = new DepthBasedMaterialProvider(...);
Scanner myScanner = new FindOnSurfaceScanner(...);
Pattern myPattern = new SolidGroundPattern();

// 2. Configure the Prop
DensityProp oreVein = new DensityProp(
    new Vector3i(8, 8, 8), // range
    myDensity,
    myMaterials,
    myScanner,
    myPattern,
    BlockMask.REPLACE_STONE,
    Material.AIR
);

// 3. The generator's scheduler uses the prop
// In the scan phase:
ScanResult result = oreVein.scan(position, worldData, workerId);

// In the place phase (if result was valid):
Prop.Context placeContext = new Prop.Context(worldData, result, workerId);
oreVein.place(placeContext);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation at Runtime:** Do not use `new DensityProp()` inside a tight generation loop. These are configuration objects meant to be created once and reused for many positions.
-   **Ignoring Dependencies:** Calling `scan` or `place` without first using `getContextDependency` to ensure the required `VoxelSpace` data is loaded and available will lead to `IndexOutOfBoundsException` or incomplete generation.
-   **State Leakage:** Do not attempt to modify the internal state of a DensityProp after construction. The injected dependencies (like Density or MaterialProvider) should also be immutable or thread-safe if they are shared across props.

## Data Pipeline
The DensityProp transforms abstract rules into concrete voxel data. The flow for a single placement is a multi-stage process involving significant internal computation.

> Flow:
> 1.  **Scan Phase:** A `Scanner` inspects the `VoxelSpace` to find a list of valid anchor `Vector3i` positions.
> 2.  **Placement Trigger:** The list of positions is passed to the `place` method.
> 3.  **Density Sampling:** For each position, `place` creates a temporary boolean 3D grid (`densitySpace`) by sampling the `Density` function over the entire volume.
> 4.  **Context Analysis:** The boolean grid is processed with vertical passes to calculate depth and proximity-to-air information for every solid voxel.
> 5.  **Material Selection:** This contextual data is fed into the `MaterialProvider`, which selects a final `Material` for each voxel.
> 6.  **Voxel Write:** The chosen `Material` is written into the main `VoxelSpace`, subject to `BlockMask` rules.

This can be visualized as:

> `Scanner` -> **DensityProp.scan** -> `PositionListScanResult` -> **DensityProp.place** -> `Density` -> *Internal Depth Analysis* -> `MaterialProvider` -> `VoxelSpace` Update

