---
description: Architectural reference for FilteredBlockFluidCondition
---

# FilteredBlockFluidCondition

**Package:** com.hypixel.hytale.server.worldgen.util.condition
**Type:** Utility

## Definition
```java
// Signature
public class FilteredBlockFluidCondition implements IBlockFluidCondition {
```

## Architecture & Concepts
FilteredBlockFluidCondition is a composite implementation of the IBlockFluidCondition interface, designed to create complex procedural generation rules through exclusion. It functions as a logical gate, combining two separate conditions—a *filter* and a *condition*—to produce a single boolean result.

Architecturally, this class embodies the Decorator pattern. It wraps a primary IBlockFluidCondition, augmenting its behavior with a preliminary filtering check. Its core purpose in the world generation engine is to prevent a feature or structure from being placed in a specific location *before* evaluating the more complex placement logic. This is a key optimization and simplification pattern, allowing developers to express rules like "place on grass, but never if the block is waterlogged."

The logical operation performed by this class can be expressed as: **NOT filter AND condition**. If the filter evaluates to true, the entire operation short-circuits and returns false, effectively masking or vetoing the placement.

## Lifecycle & Ownership
-   **Creation:** Instantiated on-demand by higher-level world generation systems, such as biome definitions, structure placement controllers, or feature generators. It is not a managed service and is not retrieved from a central registry.
-   **Scope:** Transient and short-lived. An instance typically exists only for the duration of a specific world generation pass or algorithm. Its lifetime is bound to the parent object that defines the generation rule.
-   **Destruction:** The object is lightweight and becomes eligible for garbage collection as soon as its parent generator is discarded. No explicit cleanup or resource management is necessary.

## Internal State & Concurrency
-   **State:** **Immutable**. The internal *filter* and *condition* delegates are declared as final and are injected exclusively through the constructor. Once an instance is created, its logical behavior is fixed for its entire lifetime.
-   **Thread Safety:** **Inherently Thread-Safe**. Due to its immutable design, a single instance can be safely shared and executed by multiple worker threads in a parallelized world generation environment. No external locking or synchronization is required, making it ideal for high-performance, concurrent chunk processing.

## API Surface
The public contract is minimal, consisting only of the interface method it implements.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(int block, int fluid) | boolean | O(1) | Evaluates the filtered condition. Returns false if the filter matches the input; otherwise, returns the result of the nested condition. The overall complexity depends on the composed delegates. |

## Integration Patterns

### Standard Usage
This component is used to build layered placement rules. The primary pattern is to define a prohibitive rule (the filter) and a permissive rule (the condition) and combine them.

```java
// Define a base condition: must be a grass block.
IBlockFluidCondition onGrass = (block, fluid) -> block == BlockIDs.GRASS;

// Define a filter: must NOT be a water block.
IBlockFluidCondition isWater = (block, fluid) -> block == BlockIDs.WATER;

// Combine them: place on grass, but only if the block is not water.
IBlockFluidCondition placementRule = new FilteredBlockFluidCondition(isWater, onGrass);

// During a world generation pass...
if (placementRule.eval(targetBlockId, targetFluidId)) {
    // Place the feature
}
```

### Anti-Patterns (Do NOT do this)
-   **Logic Inversion:** Do not misuse the filter and condition parameters. The first parameter is for exclusion, the second for inclusion. Swapping them will produce inverted and likely incorrect world generation results.
-   **Null Delegates:** The constructors do not perform null-safety checks. Passing a null *filter* or *condition* will not fail at instantiation but will cause a NullPointerException during the `eval` call at a later, less predictable time. Always ensure delegate conditions are non-null.
-   **Overly Complex Nesting:** Avoid nesting many FilteredBlockFluidCondition instances within each other. This creates unreadable "spaghetti logic" that is difficult to debug. For highly complex boolean chains, consider a more robust specification pattern or a dedicated rule-building utility.

## Data Pipeline
This class acts as a decision node within the broader world generation data pipeline. It consumes raw voxel data and outputs a boolean decision that controls subsequent stages.

> Flow:
> Voxel Data Stream (Block ID, Fluid ID) -> World Generator -> **FilteredBlockFluidCondition.eval()** -> Boolean Result -> Feature Placer / Structure Stamper

