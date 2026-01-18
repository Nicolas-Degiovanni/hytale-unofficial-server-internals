---
description: Architectural reference for ResolvedBlockArray
---

# ResolvedBlockArray

**Package:** com.hypixel.hytale.server.worldgen.util
**Type:** Utility / Data Structure

## Definition
```java
// Signature
public final class ResolvedBlockArray {
```

## Architecture & Concepts
The ResolvedBlockArray is an immutable, high-performance data structure designed to represent a set of valid block and fluid combinations within the server-side world generation pipeline. Its primary function is to provide an extremely fast mechanism for checking if a specific block-fluid pair is part of a pre-defined collection.

This class is a cornerstone of the **Flyweight pattern** used by the world generator to minimize memory consumption. Instead of creating duplicate objects for identical sets of block rules, the engine maintains two global, static caches:
1.  **RESOLVED_BLOCKS:** A cache for standard block resolutions.
2.  **RESOLVED_BLOCKS_WITH_VARIANTS:** A cache for resolutions that also consider block variants.

Internally, it achieves its performance by transforming the input array of BlockFluidEntry objects into a LongSet. Each block and fluid ID pair is packed into a single primitive long using MathUtil.packLong. This allows the critical `contains` operation to execute with an average time complexity of O(1), avoiding costly array iterations.

This component is not a service but a fundamental value object. Its immutability guarantees that it can be safely shared across multiple threads in the world generation engine without locks or race conditions.

## Lifecycle & Ownership
-   **Creation:** Instances are not intended to be created directly. A higher-level resolver or factory service within the world generation system is responsible for their instantiation. Upon encountering a new set of block-fluid rules, the service constructs a ResolvedBlockArray and immediately places it into one of the global static caches for reuse. The static EMPTY instance is created at class-loading time.
-   **Scope:** An instance of ResolvedBlockArray, once cached, persists for the entire lifetime of the server process. This is a deliberate design choice to maximize the effectiveness of the Flyweight pattern, assuming that world generation rules are static. Transient, uncached instances may exist briefly during a generation task but are expected to be short-lived.
-   **Destruction:** Objects are managed by the Java garbage collector. As instances stored in the static caches are globally referenced, they will not be garbage collected until server shutdown.

## Internal State & Concurrency
-   **State:** **Immutable**. The internal state, consisting of the entries array and the entrySet, is populated only once within the constructor and is never modified thereafter. All public methods are read-only.
-   **Thread Safety:** This class is **unconditionally thread-safe**. Its immutability ensures that multiple threads can read from the same instance concurrently without any risk. The static caches that store these objects are wrapped in synchronized maps (`Long2ObjectMaps.synchronize`), ensuring that access and mutation of the global caches are also thread-safe.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getEntries() | BlockFluidEntry[] | O(1) | Returns the backing array of block-fluid pairs. **Warning:** The returned array must be treated as read-only. |
| getEntrySet() | LongSet | O(1) | Returns the backing set of packed long keys for advanced queries. |
| size() | int | O(1) | Returns the number of block-fluid entries in the collection. |
| contains(block, fluidId) | boolean | O(1) | Performs a highly optimized check to see if the specified block and fluid ID pair exists in the collection. |

## Integration Patterns

### Standard Usage
This class is typically obtained from a world generation context or a resolver service, not constructed directly. Its primary use is to validate block choices during procedural generation.

```java
// Correctly obtain a ResolvedBlockArray from a worldgen resolver
ResolvedBlockArray validBlocks = worldGenContext.getValidBlockSetForBiome(plainsBiome);

// Use the contains method to make a generation decision
if (validBlocks.contains(candidateBlockId, candidateFluidId)) {
    world.setBlock(position, candidateBlockId);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ResolvedBlockArray()`. This bypasses the global caching system, defeating the memory optimization of the Flyweight pattern and putting unnecessary pressure on the garbage collector. Always use the designated factory or resolver.
-   **Modifying Returned Array:** The array returned by getEntries is the internal, canonical representation. Modifying its contents is a severe violation of the class's immutability contract and will lead to undefined, catastrophic behavior across the entire world generator, as the corrupted instance is shared globally.

## Data Pipeline
ResolvedBlockArray is not a processing stage in a pipeline; it is the resulting data artifact. Its creation pipeline is as follows:

> Flow:
> World Generation Rule Set -> Resolver Service -> **Global Cache Lookup** -> (Cache Miss) -> **new ResolvedBlockArray()** -> Populate Global Cache -> Return Shared Instance

Once created, it is used as a read-only data source for subsequent logic:

> Flow:
> Voxel Placement Algorithm -> Query **ResolvedBlockArray.contains()** -> Conditional Block Update -> World State Change

