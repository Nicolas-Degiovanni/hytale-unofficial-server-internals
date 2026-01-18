---
description: Architectural reference for BoxProp
---

# BoxProp

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props
**Type:** Transient

## Definition
```java
// Signature
public class BoxProp extends Prop {
```

## Architecture & Concepts
The BoxProp class is a concrete implementation of the Prop abstraction, representing an atomic, reusable instruction within the world generation system. Its sole responsibility is to place solid, cuboid volumes of a specific Material into the world.

Architecturally, BoxProp embodies the **Scan-and-Place** pattern, a fundamental concept in Hytale's procedural generator. This pattern decouples the read-only discovery phase from the write-only mutation phase, which is critical for determinism and parallelization.

1.  **Scan Phase:** The `scan` method uses a provided Scanner and Pattern to inspect a region of the world (a VoxelSpace) and identify all valid coordinates for placement. This phase does not modify any world data.
2.  **Place Phase:** The `place` method takes the list of valid coordinates from the scan phase and mutates the VoxelSpace, filling the box volumes at each coordinate.

The most critical architectural component is the **ContextDependency**. During construction, BoxProp pre-calculates the maximum region of world data it needs to *read* in order to make its decisions, and the maximum region it might *write* to. This dependency metadata is consumed by a higher-level scheduler, such as a StagedConveyor, to orchestrate the entire generation pipeline. The scheduler uses this information to ensure that all prerequisite world data is available before dispatching a BoxProp operation to a worker thread.

### Lifecycle & Ownership
-   **Creation:** BoxProp instances are not created dynamically during the generation loop. They are instantiated once as part of a static world generator definition, typically within a generation "Layer" or "Stage". Each instance represents a specific type of box to be placed (e.g., a 3x3x3 stone boulder, a 10x1x10 patch of grass).
-   **Scope:** An instance persists for the lifetime of the world generator configuration. It is a stateless, reusable template for a generation operation.
-   **Destruction:** The object is garbage collected when the world generator definition is unloaded or replaced.

## Internal State & Concurrency
-   **State:** A BoxProp instance is **effectively immutable**. All of its core configuration—range, material, scanner, and pattern—are set at construction and never change. Derived data, such as writeBounds_voxelGrid and contextDependency, are also calculated once in the constructor.

-   **Thread Safety:** The class is **fully thread-safe**. Its immutability allows a single instance to be safely shared and executed by multiple worker threads simultaneously without locks or synchronization. All mutable state, such as the target VoxelSpace and current position, is passed in via method arguments (e.g., Prop.Context), ensuring that concurrent operations do not interfere with each other, provided they operate on distinct VoxelSpace instances. The presence of WorkerIndexer.Id in the `scan` signature is a strong indicator of this design.

## API Surface
The public API is minimal, exposing only the core operations required by the generation scheduler.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| scan(position, materialSpace, id) | PositionListScanResult | O(N) | **Read-only.** Scans the VoxelSpace using the configured Scanner and Pattern to find valid placement locations. N is the number of voxels in the scan area. |
| place(context) | void | O(M * V) | **Write-only.** Mutates the VoxelSpace by placing boxes at positions identified by a prior scan. M is the number of positions; V is the volume of the box. |
| getContextDependency() | ContextDependency | O(1) | Returns pre-calculated data dependencies. Used by the scheduler to manage task execution. |
| getWriteBounds() | Bounds3i | O(1) | Returns the maximum possible volume this prop can affect. Used by the scheduler for chunk management and dirty-region tracking. |

## Integration Patterns

### Standard Usage
A BoxProp is intended to be used by a higher-level generation orchestrator. The orchestrator is responsible for managing the VoxelSpace, invoking the scan-and-place methods, and passing the results between them.

```java
// Conceptual usage within a generator stage
// The 'boxProp' instance is pre-configured and injected.

// 1. Execute the read-only scan phase
PositionListScanResult scanResult = boxProp.scan(chunkPosition, materialSpace, workerId);

// 2. Prepare the context for the write phase
Prop.Context placeContext = new Prop.Context(materialSpace, scanResult);

// 3. Execute the write-only place phase
boxProp.place(placeContext);
```

### Anti-Patterns (Do NOT do this)
-   **Dynamic Instantiation:** Do not use `new BoxProp()` inside a generation loop. This is highly inefficient and defeats the purpose of having reusable, pre-configured templates. Define props once at initialization.
-   **Stateful Interaction:** Do not modify the VoxelSpace between the `scan` and `place` calls for a single prop operation. The Scan-and-Place pattern assumes the world state is constant between the two phases. Modifying the state can lead to non-deterministic or corrupt results.
-   **Ignoring Dependencies:** Do not invoke `scan` or `place` without ensuring the data requirements specified by `getContextDependency` have been met. Doing so will result in `IndexOutOfBoundsException` or incomplete generation, as the prop will attempt to read data from unloaded regions.

## Data Pipeline
The flow of data for a BoxProp operation is strictly controlled by the world generation scheduler. The prop itself is a passive processor of data provided to it.

> Flow:
> Generation Scheduler -> Provides **VoxelSpace** -> **BoxProp.scan()** -> Returns **PositionListScanResult** -> Generation Scheduler -> Provides **VoxelSpace** & **PositionListScanResult** -> **BoxProp.place()** -> Mutates **VoxelSpace**

