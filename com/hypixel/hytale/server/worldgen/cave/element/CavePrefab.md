---
description: Architectural reference for CavePrefab
---

# CavePrefab

**Package:** com.hypixel.hytale.server.worldgen.cave.element
**Type:** Value Object / Transient

## Definition
```java
// Signature
public class CavePrefab implements CaveElement {
```

## Architecture & Concepts

The CavePrefab class is a fundamental data structure within the server-side procedural world generation framework. It does not represent a service or a manager, but rather a specific, immutable instruction for placing a prefabricated structure into the world during cave generation.

Its primary role is to act as a "placement manifest". It encapsulates all the necessary information for a later stage of the generation pipeline to execute the placement:

1.  **What to place:** The `WorldGenPrefabSupplier` provides access to the actual block data of the structure.
2.  **Where to place it:** The `x`, `y`, and `z` coordinates define the origin point in world space.
3.  **How to orient it:** The `PrefabRotation` specifies the desired orientation.
4.  **Under what conditions:** The `biomeMask` and `blockMask` act as predicates that must be satisfied before placement can occur, ensuring the prefab fits its environment both biologically and physically.

A critical architectural feature is the pre-computation of the world-space bounding box (`bounds`) within the constructor. By calculating the exact volume this prefab will occupy upon instantiation, higher-level systems can perform extremely fast spatial queries, intersection tests, and culling operations without needing to load and process the full, potentially large, prefab data. This makes CavePrefab a lightweight proxy for a much heavier object, which is essential for performance in a massively parallel generation environment.

## Lifecycle & Ownership

-   **Creation:** A CavePrefab is instantiated by high-level procedural generators, such as a `CaveSystemGenerator` or a `DungeonFeaturePlacer`. It is created dynamically as part of a generation pass for a specific world region.
-   **Scope:** The object's lifetime is ephemeral and strictly scoped to the world generation task that created it. It exists only in memory and is never serialized or persisted to disk.
-   **Destruction:** It is a short-lived object. Once the responsible world generation system has processed it and stamped the corresponding blocks into the chunk buffer, the CavePrefab instance is no longer referenced and becomes eligible for garbage collection. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency

-   **State:** **Immutable**. All internal fields are declared `final` and are assigned exclusively within the constructor. The object's state, including its calculated `bounds`, cannot be modified after creation. This design guarantees that a CavePrefab is a predictable and reliable data record.
-   **Thread Safety:** **Fully thread-safe**. Its immutability ensures that a CavePrefab instance can be safely passed between and read by multiple worker threads without locks or other synchronization primitives. This is a vital property for the server's parallelized world generation architecture, where different threads may need to query the object's bounds simultaneously.

## API Surface

The public API is composed entirely of non-mutating accessor methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPrefab() | WorldGenPrefabSupplier | O(1) | Returns the supplier for the raw prefab data. |
| getRotation() | PrefabRotation | O(1) | Returns the specified orientation for placement. |
| getBounds() | IWorldBounds | O(1) | Returns the pre-calculated, world-space bounding box. |
| getX() | int | O(1) | Returns the world-space X coordinate of the placement origin. |
| getBiomeMask() | IIntCondition | O(1) | Returns the conditional mask for biome validation. |

## Integration Patterns

### Standard Usage

CavePrefab is not intended for direct use by most game logic. It is an internal component of the world generation pipeline. A generator creates a collection of CaveElement instances, including CavePrefabs, which are then consumed by a placement engine.

```java
// A generator produces a list of elements to be placed.
List<CaveElement> elements = caveGenerator.generateElementsForRegion(region);

// A placement system iterates and processes each element.
for (CaveElement element : elements) {
    if (element instanceof CavePrefab) {
        CavePrefab prefabPlacement = (CavePrefab) element;

        // Check bounds and conditions before committing to placement.
        if (placementEngine.canPlace(prefabPlacement.getBounds(), prefabPlacement.getConfiguration())) {
            placementEngine.stampPrefab(prefabPlacement);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **State Modification:** Do not attempt to modify a CavePrefab's state after creation using reflection. This violates its immutability contract and will cause desynchronization between its `bounds` and the actual prefab data, leading to severe world generation artifacts like corrupted or floating structures.
-   **Caching and Re-use:** Do not cache and re-use a CavePrefab instance for a different location. Each instance is tied to a specific set of coordinates which are baked into its pre-calculated bounding box. Re-using it would result in incorrect spatial calculations.
-   **Manual Instantiation:** Avoid creating a CavePrefab manually. These objects should be the output of a trusted procedural algorithm that has already performed the necessary checks to propose a valid placement location.

## Data Pipeline

CavePrefab serves as a data container that flows from high-level layout generation to low-level block placement.

> Flow:
> Procedural Layout Algorithm -> **new CavePrefab(...)** -> Collection of CaveElement -> Placement Engine -> Prefab Data is read via `getPrefab()` -> Blocks written to World Buffer

