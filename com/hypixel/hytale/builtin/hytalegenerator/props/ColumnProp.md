---
description: Architectural reference for ColumnProp
---

# ColumnProp

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props
**Type:** Transient / Data Object

## Definition
```java
// Signature
public class ColumnProp extends Prop {
```

## Architecture & Concepts
The ColumnProp is a fundamental component within the procedural world generation engine, specifically designed to represent a vertical arrangement of blocks. It acts as a self-contained, configurable "recipe" for placing features like pillars, tree trunks, stalactites, or any other column-like structure into the world.

This class encapsulates not only the definition of the column—its height and the materials at each vertical position—but also the logic for its placement. It bridges the declarative data (the blocks) with the procedural logic (scanning for valid locations and applying placement rules).

Its primary role is to be configured once during the setup of a world generator (e.g., for a specific biome) and then be executed repeatedly by the generation pipeline across many different world chunks. A key performance optimization is the pre-computation of rotated block variants in the constructor, which avoids costly material lookups during the performance-critical `place` phase.

It relies on several collaborator components:
-   **Scanner:** Defines the algorithm for finding valid anchor points on the terrain where a column can be placed.
-   **Directionality:** Determines the correct rotation of the column based on the surrounding context, allowing for more natural and varied placements.
-   **BlockMask:** A ruleset that governs which existing blocks in the world can be replaced by the column's materials.
-   **MaterialCache:** A shared service used to efficiently retrieve pre-rotated material definitions.

## Lifecycle & Ownership
-   **Creation:** Instantiated during the configuration phase of a world generator. It is not created on-the-fly during the main generation loop. The constructor requires a complete definition, including vertical positions, materials, a scanner, and placement rules.
-   **Scope:** The object's lifetime is tied to the world generator configuration that defines it. It typically persists within a collection of available props for a biome or generation stage.
-   **Destruction:** Managed by the Java garbage collector. It is eligible for collection once the world generator or the specific stage it belongs to is no longer in use. There are no explicit destruction methods.

## Internal State & Concurrency
-   **State:** The ColumnProp is stateful but effectively immutable after its constructor completes. All defining characteristics, including the arrays of materials for each of the four cardinal rotations (`blocks0`, `blocks90`, `blocks180`, `blocks270`), are stored in final fields. This pre-calculation of rotated states is a critical design choice for performance.
-   **Thread Safety:** This class is designed to be thread-safe for use in a parallelized world generation system.
    -   Its internal state is read-only after construction, allowing multiple threads to call its methods concurrently without risk of data corruption.
    -   The `scan` and `place` methods operate on method-scoped data structures like VoxelSpace and Prop.Context. The responsibility for ensuring thread-safe access to the underlying VoxelSpace data lies with the VoxelSpace implementation itself, not this class.
    -   The presence of the WorkerIndexer.Id parameter in the `scan` method is a clear indicator that this class is intended to be used by a multi-threaded worker system.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| scan(position, materialSpace, id) | ScanResult | O(N) | Executes the configured Scanner to find all valid anchor points and their required rotations within a given region. N is the number of voxels evaluated by the Scanner. |
| place(context) | void | O(M * H) | Modifies the VoxelSpace by placing the column at all valid locations found by a prior scan. M is the number of positions in the ScanResult, and H is the height of the column. |
| getContextDependency() | ContextDependency | O(1) | Returns the data dependency contract, specifying the read/write region required by this prop. Essential for parallel task schedulers. |
| getWriteBounds() | Bounds3i | O(1) | Returns the pre-calculated maximum bounds this prop can write to, relative to an anchor point. |

## Integration Patterns

### Standard Usage
The ColumnProp is intended to be used by a higher-level generation orchestrator, such as a `StagedConveyor`. The orchestrator manages a list of props and executes them in stages across world chunks.

```java
// Orchestrator retrieves a pre-configured ColumnProp
ColumnProp pillarProp = biome.getProp("pillar");

// In a worker thread for a specific world region...
VoxelSpace chunkView = world.getVoxelSpaceForChunk(chunkPos);
WorkerIndexer.Id workerId = threadPool.getWorkerId();

// Phase 1: Scan for placement opportunities
ScanResult locations = pillarProp.scan(chunkPos.getMin(), chunkView, workerId);

// Phase 2: Apply the changes
if (!locations.isEmpty()) {
    Prop.Context placeContext = new Prop.Context(locations, chunkView, workerId);
    pillarProp.place(placeContext);
}
```

### Anti-Patterns (Do NOT do this)
-   **Instantiation in a Loop:** Do not create new ColumnProp instances inside the main generation loop. The constructor performs significant pre-calculation and is designed to be called only once during initialization.
-   **Ignoring ContextDependency:** Failing to use the information from `getContextDependency` in a parallel generator will lead to severe race conditions and visual artifacts at chunk boundaries. The orchestrator *must* respect these bounds to ensure workers do not overwrite each other's results.
-   **External State Modification:** Do not attempt to modify the internal state of a ColumnProp after it has been constructed. It is designed as an immutable data object.

## Data Pipeline
The flow of data through a ColumnProp involves a two-phase process: scanning and placing.

> Flow:
> Generator Orchestrator -> **ColumnProp.scan()** -> (Internal Scanner & Directionality query VoxelSpace) -> RotatedPositionsScanResult -> Generator Orchestrator -> **ColumnProp.place()** -> (Internal BlockMask checks VoxelSpace) -> Modified VoxelSpace

