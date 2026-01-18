---
description: Architectural reference for PondFillerProp
---

# PondFillerProp

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props.filler
**Type:** Transient

## Definition
```java
// Signature
public class PondFillerProp extends Prop {
```

## Architecture & Concepts
The PondFillerProp is a specialized component within the procedural world generation engine, responsible for creating bodies of fluid, such as ponds or puddles. It operates as a "feature prop," a self-contained algorithm that is applied to the world during a specific generation stage.

Architecturally, this class implements a sophisticated two-phase **scan-then-place** pattern. This separation is critical for parallel world generation, preventing race conditions where one prop's placement might interfere with another's initial detection logic.

The core of its functionality is a bounded flood-fill simulation. Unlike a simple block replacement, it intelligently identifies a contained basin or depression in the terrain, determines which air blocks are sealed by solid ground, and fills only those blocks. This prevents fluid from "leaking" out of unnatural boundaries and results in more organic-looking features.

PondFillerProp heavily utilizes the **Strategy Pattern** for configurability:
*   **Scanner & Pattern:** These external strategies define *where* and *how* to find candidate locations for ponds. The PondFillerProp itself does not decide the initial placement trigger; it only executes the filling logic once a location is provided.
*   **MaterialSet:** A strategy for defining what constitutes a "solid" or "container" block. This allows the same filler logic to be used for water ponds in dirt, lava pools in stone, etc.
*   **MaterialProvider:** A strategy that determines the exact material to fill the basin with, allowing for variations like clear water, murky water, or other fluids.

## Lifecycle & Ownership
- **Creation:** PondFillerProp is instantiated and configured by a higher-level world generation orchestrator, typically during the setup of a biome or a specific generation layer. It is not intended for direct instantiation by client code. Its dependencies (Scanner, MaterialProvider, etc.) are injected at construction.

- **Scope:** The lifetime of a PondFillerProp instance is typically tied to a single, cohesive world generation stage. It is created, used to process potentially thousands of candidate locations across multiple worker threads, and then becomes eligible for garbage collection once the stage is complete. It does not persist between sessions or major generation phases.

- **Destruction:** The object is managed by the Java garbage collector. There are no native resources or manual cleanup steps required.

## Internal State & Concurrency
- **State:** The internal state of a PondFillerProp instance is **effectively immutable** after construction. All configuration fields such as boundingMin, boundingMax, solidSet, and material providers are final and are set only once in the constructor. The constructor defensively clones incoming Vector3i objects to prevent external mutation.

- **Thread Safety:** This class is **thread-safe** and designed for concurrent execution. The primary methods, scan and place, do not modify the internal state of the PondFillerProp instance. All mutable state required for the flood-fill algorithm, such as the integer mask, is created locally within the method scope (e.g., `renderFluidBlocks`). This re-entrant design allows a single configured instance to be safely shared and used by multiple world generation worker threads operating on different sections of the world.

## API Surface
The public contract is designed for integration with a world generation orchestrator.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| scan(position, materialSpace, id) | FillerPropScanResult | O(N) | **Phase 1:** Scans for a valid placement and simulates the fill. N is the number of voxels in the bounding box. Returns a result object containing the coordinates to be filled. |
| place(context) | void | O(M) | **Phase 2:** Commits the changes to the world. M is the number of fluid blocks identified in the scan phase. Throws ClassCastException if context contains an invalid scan result. |
| getContextDependency() | ContextDependency | O(1) | Provides dependency information to the generation scheduler, indicating the data region this prop needs to read. |
| getWriteBounds() | Bounds3i | O(1) | Provides bounds information to the generation scheduler, indicating the data region this prop may write to. |

## Integration Patterns

### Standard Usage
The PondFillerProp is used in a two-phase process managed by a generation scheduler. The scheduler first calls `scan` to get a result, and later passes that result to the `place` method.

```java
// Conceptual example within a world generator
// 1. Configuration
PondFillerProp waterPond = new PondFillerProp(
    new Vector3i(-5, -3, -5),
    new Vector3i(5, 0, 5),
    biome.getSolidMaterials(),
    new StaticMaterialProvider(Material.WATER),
    new FindDepressionScanner(),
    new BasicPattern()
);

// 2. Scan Phase (executed by a worker)
FillerPropScanResult scanResult = waterPond.scan(candidatePosition, worldVoxelSpace, workerId);

// 3. Place Phase (executed by a worker after scan phase completes)
// The context object is provided by the generation framework.
Prop.Context placeContext = new Prop.Context(scanResult, worldVoxelSpace, workerId, ...);
waterPond.place(placeContext);
```

### Anti-Patterns (Do NOT do this)
- **Stateful MaterialProviders:** Avoid using a MaterialProvider with internal state that is not thread-safe. Since a single PondFillerProp may be used by multiple threads, any shared dependency must also be thread-safe.

- **Incorrect SolidSet Configuration:** Providing a MaterialSet that incorrectly defines solid blocks is the most common source of errors. If the set is too permissive, ponds will leak. If it is too restrictive, valid basins will not be filled.

- **Overlapping Bounds:** Configuring the bounding box to be excessively large can cause severe performance degradation, as the complexity of the `scan` method is directly tied to the volume of the bounding box.

## Data Pipeline
The flow of data for generating a single pond is a multi-step process orchestrated by the world generator.

> Flow:
> World Generator identifies candidate chunk -> Provides `VoxelSpace` -> **PondFillerProp.scan** -> Internal flood-fill simulation using a temporary bitmask -> Generates `FillerPropScanResult` (a list of coordinates) -> World Generator schedules placement -> **PondFillerProp.place** -> Iterates coordinates -> Fetches `Material` from `MaterialProvider` -> Writes material to `VoxelSpace`

