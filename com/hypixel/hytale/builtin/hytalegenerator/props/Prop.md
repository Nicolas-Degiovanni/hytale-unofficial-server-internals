---
description: Architectural reference for Prop
---

# Prop

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props
**Type:** Abstract Base Class / Template

## Definition
```java
// Signature
public abstract class Prop {
```

## Architecture & Concepts
The Prop class is an abstract contract that defines a single, placeable feature within the world generation system. It represents discrete objects such as trees, rocks, bushes, or small structures. It is a foundational component of the procedural decoration layer, which operates after the primary terrain shape has been established.

The core architectural pattern is a **two-phase commit system** for world modification, embodied by the `scan` and `place` methods.

1.  **Scan Phase:** A Prop first inspects a target location in the world to determine if it *can* be placed. This is a read-only operation that checks for conditions like valid ground material, sufficient space, and proximity to other features. This phase produces a `ScanResult` object containing the necessary data for the next phase.
2.  **Place Phase:** If the scan is successful, the Prop is later instructed to perform the actual placement. This is a write operation that modifies the `VoxelSpace` (placing blocks) and the `EntityContainer` (spawning entities).

This two-phase design is critical for enabling concurrent world generation. It allows the system to speculatively check many potential placements across multiple worker threads without causing write conflicts. The actual write operations (place) can then be scheduled and executed in a controlled manner, respecting data dependencies.

### Lifecycle & Ownership
-   **Creation:** Instances of Prop subclasses are created dynamically by higher-level generator systems, such as a `PropPlacer` or a biome-specific decorator. They are intended to be lightweight, transient objects. The static factory `noProp` provides a Null Object Pattern implementation for cases where no operation is required.
-   **Scope:** The lifecycle of a Prop instance is extremely short. It is typically created to evaluate and execute a single potential placement within a specific world generation task. It does not persist beyond the scope of that task.
-   **Destruction:** Instances are eligible for garbage collection immediately after their `place` method is called, or after their `scan` method returns a negative result. They hold no persistent state within the engine.

## Internal State & Concurrency
-   **State:** The abstract Prop class is stateless. Subclasses may contain immutable configuration data (e.g., the dimensions of a schematic or the types of materials to use), but they must not hold mutable state related to a specific world location. All necessary world state is passed into the `scan` and `place` methods via parameters.
-   **Thread Safety:** The Prop contract is designed for a highly concurrent environment. The methods are expected to be called from a worker pool, as indicated by the `WorkerIndexer.Id` parameter. Thread safety is achieved through a data partitioning strategy orchestrated by the world generator:
    -   Prop instances themselves are inherently thread-safe due to their stateless nature.
    -   The `VoxelSpace` and `EntityContainer` objects passed into the methods are not globally thread-safe. The calling system is responsible for ensuring that no two worker threads are attempting to write to the same region of world data simultaneously. The `getWriteBounds` and `getContextDependency` methods provide the scheduler with the necessary information to prevent such conflicts.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| scan(position, materialSpace, id) | ScanResult | Varies | **Read-only.** Checks if the prop can be placed at the given position. Complexity depends on the size and logic of the prop. |
| place(context) | void | Varies | **Write-only.** Modifies the world state based on a prior successful scan. Throws exceptions on logical errors. |
| getContextDependency() | ContextDependency | O(1) | Returns the bounding box of world data that must be readable during the `scan` and `place` phases. |
| getWriteBounds() | Bounds3i | O(1) | Returns the bounding box of world data that this prop may modify during the `place` phase. Critical for conflict detection. |
| noProp() | static Prop | O(1) | Factory method for a singleton-like Null Object that performs no action. |

## Integration Patterns

### Standard Usage
A Prop is never used in isolation. It is always driven by a generator or scheduler which manages the two-phase lifecycle. The caller is responsible for interpreting the `ScanResult` and constructing the `Prop.Context` for placement.

```java
// Conceptual example within a world generator
Prop treeProp = new OakTreeProp(/* config */);
Vector3i targetPosition = new Vector3i(100, 64, 200);

// Phase 1: Scan
ScanResult result = treeProp.scan(targetPosition, world.getVoxelSpace(), workerId);

// Phase 2: Place (often in a separate, scheduled task)
if (!result.isNegative()) {
    Prop.Context context = new Prop.Context(
        result,
        world.getVoxelSpace(),
        world.getEntityBuffer(),
        workerId,
        0.5
    );
    treeProp.place(context);
}
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Implementations:** Subclasses of Prop must not store mutable state. Storing a reference to the `VoxelSpace` or caching results between calls will lead to severe concurrency bugs.
-   **Invalid Lifecycle:** Calling `place` without a corresponding successful `scan` is a violation of the contract. The `Prop.Context` requires a valid `ScanResult`, and bypassing this check will lead to undefined behavior or crashes.
-   **Ignoring Bounds:** The bounds returned by `getContextDependency` and `getWriteBounds` are not optional. The generation scheduler relies on this information to manage data dependencies and prevent race conditions between worker threads. An incorrect implementation will corrupt world data.

## Data Pipeline
The Prop acts as a processor within the larger world generation pipeline. Its flow is defined by the separation of read and write operations.

> Flow:
> Generator selects a Prop -> **Prop.scan()** reads from `VoxelSpace` -> `ScanResult` is produced -> Scheduler collects results -> Scheduler creates `Prop.Context` -> **Prop.place()** writes to `VoxelSpace` & `EntityContainer` -> Modified world data

