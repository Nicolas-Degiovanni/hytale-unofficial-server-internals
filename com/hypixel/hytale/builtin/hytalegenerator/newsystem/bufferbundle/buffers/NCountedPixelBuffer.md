---
description: Architectural reference for NCountedPixelBuffer
---

# NCountedPixelBuffer<T>

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.bufferbundle.buffers
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class NCountedPixelBuffer<T> extends NPixelBuffer<T> {
```

## Architecture & Concepts
The NCountedPixelBuffer is a specialized, memory-optimized data structure designed to store a small, fixed-size 3D grid of data, typically for world generation tasks like biome or material placement. Its primary architectural feature is a state machine that dynamically changes the internal storage strategy to minimize memory footprint.

This class operates on a fixed 8x1x8 grid, as defined by the static constant SIZE_VOXEL_GRID. It is not a general-purpose buffer for arbitrary sizes.

The core optimization revolves around its internal state:
-   **EMPTY:** The initial state. The buffer contains no data and consumes minimal memory.
-   **SINGLE_VALUE:** If the entire 8x1x8 grid is filled with a single, homogeneous value, the buffer avoids allocating an array. It stores only that one value, resulting in a significant memory saving.
-   **ARRAY:** When a second, different value is written to the buffer, the internal state transitions. The buffer allocates a full 64-element array and copies the original single value into all elements before setting the new, different value.

The "Counted" aspect of its name refers to its secondary function: tracking the list of unique values present within the buffer. This is maintained in the ARRAY state and is likely used by higher-level systems for palette generation, data analysis, or material list compilation for a given region.

This component is a fundamental building block in the world generator's data pipeline, acting as a temporary, mutable workspace for a small volume of the world before the data is committed to a larger, more permanent structure.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly via its constructor: `new NCountedPixelBuffer<>(type)`. It is typically created by a higher-level generator or a buffer management system that orchestrates a generation pass.
-   **Scope:** The object's lifetime is short and tied to a specific, localized generation task. It holds transient data for a single 8x1x8 volume and is intended to be discarded after its contents have been processed or copied.
-   **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as the owning generation process releases its reference. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
-   **State:** Highly mutable. The internal representation of the stored data (a single reference vs. a full array) changes dynamically based on calls to setPixelContent. It also caches a list of unique entries when in the ARRAY state.
-   **Thread Safety:** **This class is not thread-safe.** All internal state transitions and field assignments are non-atomic. Concurrent calls to setPixelContent from multiple threads will lead to race conditions, data corruption, and an inconsistent internal state. All access to a single NCountedPixelBuffer instance must be externally synchronized or confined to a single thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPixelContent(position) | T | O(1) | Retrieves the value at the specified grid coordinate. |
| setPixelContent(position, value) | void | Amortized O(1) | Sets the value at a coordinate. **Warning:** The first call that introduces a second unique value triggers a state transition, incurring an O(N) cost for array allocation and filling, where N is 64. |
| getUniqueEntries() | List<T> | O(1) | Returns a list of all unique values currently in the buffer. The list is pre-computed and cached. |
| copyFrom(sourceBuffer) | void | O(N) | Performs a deep copy of the state from another buffer. Complexity depends on the source buffer's state (O(1) for SINGLE_VALUE, O(N) for ARRAY). |
| getMemoryUsage() | Report | O(1) | Provides an estimate of the memory consumed by the buffer's internal structures. |

## Integration Patterns

### Standard Usage
The intended pattern is to create a buffer, populate it during a generation stage, and then read its data for composition into a larger world structure. The state-based optimization is handled transparently.

```java
// A generator populates a buffer for a small world region.
Class<Biome> biomeType = Biome.class;
NCountedPixelBuffer<Biome> biomeBuffer = new NCountedPixelBuffer<>(biomeType);

// This first write sets the state to SINGLE_VALUE.
biomeBuffer.setPixelContent(new Vector3i(0, 0, 0), Biomes.FOREST);

// ... logic fills the rest of the buffer ...

// This write may trigger the transition to the ARRAY state if the biome is different.
// This is the most expensive operation on the buffer.
biomeBuffer.setPixelContent(new Vector3i(4, 0, 4), Biomes.RIVER);

// A composer can then read the final data or unique entries.
List<Biome> presentBiomes = biomeBuffer.getUniqueEntries();
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Never call setPixelContent or copyFrom on the same instance from multiple threads without external locking. This will corrupt the buffer's internal state.
-   **Ignoring Transition Cost:** Do not repeatedly create and destroy buffers in a performance-critical loop where the data pattern frequently switches between one and two unique values. The cost of the `switchFromSingleValueToArray` operation can become a performance bottleneck if triggered excessively.
-   **Out-of-Bounds Access:** The class asserts that all position vectors are within the 8x1x8 grid. Providing coordinates outside these bounds is a programming error and will fail in development environments.

## Data Pipeline
NCountedPixelBuffer acts as an intermediate container within a larger data generation pipeline. It does not produce or consume data from external systems directly.

> Flow:
> World Generation Algorithm -> `setPixelContent` -> **NCountedPixelBuffer (State: SINGLE_VALUE or ARRAY)** -> `getPixelContent` / `getUniqueEntries` -> World Chunk Composer / Palette Optimizer

