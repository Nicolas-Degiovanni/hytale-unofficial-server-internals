---
description: Architectural reference for PrefabProp
---

# PrefabProp

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props.prefab
**Type:** Transient

## Definition
```java
// Signature
public class PrefabProp extends Prop {
```

## Architecture & Concepts
The PrefabProp is a fundamental component of the Hytale world generator, responsible for placing pre-designed structures, known as prefabs, into the game world. It is a concrete implementation of the abstract Prop class, which represents a single, configurable world generation rule.

Unlike a simple copy-paste operation, PrefabProp encapsulates a sophisticated placement pipeline that provides variety, environmental awareness, and compositional power. It acts as the primary mechanism for populating the world with everything from small decorative elements like rocks and trees to large, complex dungeons and villages.

Key architectural concepts include:

-   **Prefab Pool:** At its core, a PrefabProp manages a WeightedMap of prefab variations. This allows the generator to select from a pool of different but thematically related structures (e.g., several different tree models for a forest) with configurable probabilities, which is essential for creating natural-looking and non-repetitive environments.

-   **Scanning & Directionality:** Placement is not random. The PrefabProp uses an injected Scanner object to analyze the target VoxelSpace and identify valid locations based on terrain patterns. A corresponding Directionality object then determines the optimal rotation for the prefab at that location, allowing structures to align with cliffs, paths, or other features.

-   **Terrain Molding:** A critical feature for seamless integration is terrain molding. When enabled, the PrefabProp can vertically deform a structure to match the underlying terrain contours. It performs a vertical scan using a dedicated moldingScanner to calculate height offsets, ensuring that foundations sit correctly on uneven ground rather than floating or being buried.

-   **Composition via Child Props:** Prefabs can be nested. A PrefabProp is responsible for parsing child prefab definitions within its primary prefab. It recursively constructs and manages child PrefabProp instances, placing them relative to the parent's final position and rotation. This allows for the creation of highly complex, modular structures from smaller, reusable parts.

-   **Material Masking:** To prevent destructive placements, the prop operates with a BlockMask. This mask defines rules about which existing materials in the world can be overwritten. This is crucial for safely placing a dungeon into a mountain without destroying rare ore veins, or for layering props on top of each other.

## Lifecycle & Ownership
-   **Creation:** A PrefabProp is instantiated by a higher-level world generation orchestrator, typically a factory or parser that reads zone configuration files. All dependencies—such as the prefab data, scanner logic, and material masks—are injected via its constructor. During construction, it recursively creates and configures all necessary child PrefabProp instances.

-   **Scope:** An instance of PrefabProp represents a single, immutable generation rule. It is not a global singleton. Its lifetime is tied to the generation process of a specific world region (e.g., a zone or a set of chunks). It persists as long as that rule is needed and is discarded once the region's generation is complete.

-   **Destruction:** The object is managed by the Java garbage collector. There are no manual cleanup or `dispose` methods. Once the world generator releases all references to a PrefabProp instance, it becomes eligible for garbage collection.

## Internal State & Concurrency
-   **State:** The PrefabProp is effectively **immutable** after its constructor completes. All configuration and dependencies are stored in final fields. Complex data, such as the prop's potential write bounds and child prop relationships, are pre-calculated during instantiation. This design is intentional and critical for safe parallel execution.

-   **Thread Safety:** This class is **thread-safe** and designed for concurrent execution. The `scan` and `place` methods are expected to be called by multiple worker threads simultaneously on different sections of the world.
    -   It contains no mutable instance state that is modified by its core methods. All state changes are applied to the `VoxelSpace` and `EntityContainer` objects passed as method parameters.
    -   Randomness is handled deterministically. The internal `SeedGenerator` is used to create a `Random` object seeded with the world coordinates of the placement. This ensures that the same prefab is chosen and placed identically every time for a given world seed, even when executed by different threads in a different order.

## API Surface
The public API is defined by its parent `Prop` class, focusing on the two major steps of world generation: scanning and placement.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| scan(position, materialSpace, id) | ScanResult | O(N) | Scans the provided VoxelSpace using the injected Scanner to find all valid placement locations and orientations. N is the size of the scan area. |
| place(context) | void | O(M * V) | Places the selected prefab into the VoxelSpace at all positions identified in the ScanResult. M is the number of positions, and V is the number of voxels in the prefab. |
| getContextDependency() | ContextDependency | O(1) | Returns pre-calculated spatial requirements, including the read/write bounding boxes needed for this prop to function. |
| getWriteBounds() | Bounds3i | O(1) | Returns the maximum possible bounding box that this prop might modify. |

## Integration Patterns

### Standard Usage
The PrefabProp is designed to be used within a staged world generation pipeline. A controlling system instantiates it with the desired configuration and then invokes its methods on a target world region.

```java
// 1. A generator system configures and creates the Prop
PrefabProp treeProp = createTreePropFromConfig(zoneConfig);

// 2. For a given world space, it first scans for locations
ScanResult potentialTreeLocations = treeProp.scan(origin, worldVoxelSpace, workerId);

// 3. It then creates a context and executes the placement
Prop.Context placeContext = new Prop.Context(potentialTreeLocations, worldVoxelSpace, entityBuffer, workerId);
treeProp.place(placeContext);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not manually construct a PrefabProp using `new`. Its dependency graph is complex and should be managed by a dedicated factory or configuration loader to ensure all components like MaterialCache and SeedBox are correctly provided.

-   **State Modification:** Do not attempt to modify the internal state of a PrefabProp after construction via reflection or other means. Its immutability is key to ensuring deterministic and thread-safe world generation.

-   **Ignoring ScanResult:** Do not call `place` with a manually created or empty `ScanResult`. The `place` method is tightly coupled to the output of its corresponding `scan` method, which provides the necessary position and rotation data.

## Data Pipeline
The PrefabProp transforms declarative configuration into concrete voxel and entity data within the world.

> **Flow:**
> World Generation Configuration -> **PrefabProp Factory** (Instantiation)
>
> **VoxelSpace** (Input Terrain) -> **PrefabProp.scan()** -> **ScanResult** (List of potential RotatedPositions)
>
> **ScanResult** + **VoxelSpace** -> **PrefabProp.place()**
>
> 1.  Selects a `PrefabBuffer` from its weighted pool using a position-derived seed.
> 2.  (If molding) Scans terrain vertically to create a height-offset map.
> 3.  Iterates through every block and entity in the `PrefabBuffer`.
> 4.  Applies rotation and molding offsets to calculate world coordinates.
> 5.  Checks `BlockMask` to see if placement/replacement is allowed.
> 6.  Writes final `Material` to the output `VoxelSpace`.
> 7.  Adds entities to the output `EntityContainer`.
> 8.  Recursively invokes `place()` for all child props.
>
> -> **Modified VoxelSpace** + **Populated EntityContainer** (Final Output)

