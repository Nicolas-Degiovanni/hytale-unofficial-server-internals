---
description: Architectural reference for UnionProp
---

# UnionProp

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props
**Type:** Transient

## Definition
```java
// Signature
public class UnionProp extends Prop {
```

## Architecture & Concepts

The UnionProp class is a structural component within the world generation framework, implementing the **Composite** design pattern. Its primary function is to group an ordered collection of child Prop objects and treat them as a single, unified Prop. This allows complex world features, composed of multiple smaller elements (e.g., a tree with roots, a boulder, and surrounding foliage), to be managed, scanned, and placed as one atomic operation.

It acts as a high-level orchestrator, delegating the core generation logic to its children. A critical architectural function of UnionProp is the aggregation of spatial requirements. During construction, it calculates the maximum possible read and write bounds from all its children. This unified boundary, exposed via the ContextDependency, allows the parent generation system to perform efficient spatial queries and allocate worker tasks without needing to inspect each child Prop individually.

The class operates within the generator's standard two-phase process:
1.  **Scan Phase:** The `scan` method is called to determine if the composite feature can be placed at a given location. It iterates through all child Props, invokes their respective `scan` methods, and collects the results into a specialized ChainedScanResult object.
2.  **Place Phase:** If the scan is successful, the `place` method is called. It unpacks the ChainedScanResult and dispatches a `place` call to each child Prop, providing it with its corresponding ScanResult from the first phase.

## Lifecycle & Ownership

-   **Creation:** A UnionProp is instantiated by higher-level generator logic, such as a biome or feature generator. It is constructed with a complete, pre-determined list of child Props that constitute the composite feature.
-   **Scope:** The object is task-scoped and short-lived. Its lifetime is confined to a specific world generation pass for a given region. It is not a shared service and should not be persisted across distinct generation tasks.
-   **Destruction:** The instance becomes eligible for garbage collection as soon as the generation pass that created it completes. There are no manual cleanup or disposal mechanisms required.

## Internal State & Concurrency

-   **State:** The UnionProp is stateful, but its configuration is immutable after construction. The internal list of child Props and the calculated spatial bounds (ContextDependency and writeBounds_voxelGrid) are finalized in the constructor and are not modified during the object's lifetime.
-   **Thread Safety:** This class is **not thread-safe**. The `scan` and `place` methods are designed to be executed by a single worker thread at a time. They operate on shared, mutable data structures like VoxelSpace. The world generation engine, likely a conveyor-based system, is responsible for partitioning work and ensuring that a single UnionProp instance is not accessed concurrently. The presence of the WorkerIndexer.Id parameter in the `scan` method is a clear indicator that the calling system manages parallelism.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| scan(position, materialSpace, id) | ScanResult | O(N) | Delegates the scan operation to each of its N child Props. Aggregates all results into an internal ChainedScanResult. |
| place(context) | void | O(N) | Delegates the placement operation to each of its N child Props. Throws IllegalArgumentException if the context does not contain a valid ChainedScanResult from a prior `scan` call. |
| getContextDependency() | ContextDependency | O(1) | Returns a clone of the pre-calculated, aggregated spatial dependencies of all child Props. |
| getWriteBounds() | Bounds3i | O(1) | Returns the pre-calculated, aggregated write bounds for all child Props. |

## Integration Patterns

### Standard Usage

UnionProp is used to compose complex features from simpler ones. The developer defines a list of child Props and encapsulates them within a UnionProp to be handled by the generator as a single entity.

```java
// Example: Creating a "Forest Clearing" feature
List<Prop> clearingProps = new ArrayList<>();
clearingProps.add(new LargeOakTreeProp());
clearingProps.add(new BoulderProp(BoulderSize.MEDIUM));
clearingProps.add(new FernPatchProp());

// The UnionProp treats the tree, boulder, and ferns as one feature
Prop forestClearing = new UnionProp(clearingProps);

// The world generator can now use 'forestClearing' like any other single Prop
generator.addFeature(forestClearing);
```

### Anti-Patterns (Do NOT do this)

-   **Post-Construction Modification:** Do not attempt to modify the internal list of Props after the UnionProp has been constructed. The class is not designed for dynamic composition and relies on the immutable state established during instantiation.
-   **Mismatched Scan and Place Context:** The ScanResult generated by the `scan` method is an internal contract for the `place` method of the *same instance*. Passing a ScanResult from a different Prop or a different UnionProp instance to the `place` method will cause a runtime crash.
-   **Direct Instantiation for Single Props:** Using a UnionProp to wrap a single child Prop is redundant and adds unnecessary overhead. Use the child Prop directly in such cases.

## Data Pipeline

The flow of data and control through a UnionProp instance follows a distinct two-phase pattern orchestrated by the world generator.

> **Scan Phase Flow:**
> Generator invokes `scan` -> **UnionProp** iterates its children -> Child Prop 1 `scan` called -> Child Prop 2 `scan` called -> ... -> **UnionProp** aggregates results into a ChainedScanResult -> ChainedScanResult returned to Generator

> **Place Phase Flow:**
> Generator invokes `place` with Context -> **UnionProp** validates and casts ScanResult -> **UnionProp** iterates its children -> Child Prop 1 `place` called with its specific result -> Child Prop 2 `place` called with its specific result -> ... -> VoxelSpace is mutated by children

