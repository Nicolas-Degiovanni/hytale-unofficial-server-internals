---
description: Architectural reference for DoubleMap
---

# DoubleMap

**Package:** com.hypixel.hytale.server.worldgen.climate.util
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class DoubleMap {
```

## Architecture & Concepts
The DoubleMap is a low-level, performance-oriented data structure designed to represent a 2D grid of double-precision floating-point values. It serves as a fundamental building block within the server-side world generation system, primarily for storing and manipulating spatial data like temperature, humidity, or elevation maps.

Architecturally, it is a direct abstraction over a single-dimensional `double` array. This design choice is deliberate to maximize performance and memory locality. By mapping 2D coordinates to a 1D index, it avoids the overhead and potential memory fragmentation of a jagged array (`double[][]`), which is critical for the computationally intensive loops found in procedural generation algorithms.

The class provides no high-level features such as interpolation, filtering, or serialization. It is a raw data container, optimized for high-speed read and write access, acting as a "scratchpad" for various stages of the climate simulation and terrain generation pipeline.

## Lifecycle & Ownership
- **Creation:** A DoubleMap is instantiated directly via its constructor (`new DoubleMap(width, height)`). It is not managed by a dependency injection container or service locator. Ownership is held exclusively by the system that creates it, typically a world generation task or a climate processor.

- **Scope:** The object's lifetime is intentionally short and task-specific. It typically exists only for the duration of a single, focused computation, such as generating the climate data for one world chunk.

- **Destruction:** The object is eligible for garbage collection as soon as the owning generation process completes and all references to the instance are dropped. There are no manual cleanup or `close` methods required.

## Internal State & Concurrency
- **State:** The internal state is highly mutable. While the dimensions (`width` and `height`) are immutable after construction, the core `values` array is designed for frequent modification. The `clear` method resets the state by filling the array with a default value of -1.0, which often signifies uninitialized or invalid data within the world generation context.

- **Thread Safety:** **This class is not thread-safe.** It contains no internal locks, synchronization, or atomic operations. All methods perform direct, unsynchronized array access. This is a deliberate design decision to eliminate performance overhead.

    **WARNING:** Concurrent writes to a shared DoubleMap instance from multiple threads will lead to race conditions and corrupt data. Any multi-threaded access must be managed externally, either by providing each thread with its own DoubleMap instance or by implementing an explicit locking mechanism around a shared instance.

## API Surface
The public API is minimal, focusing on direct data manipulation and coordinate translation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| at(int x, int y) | double | O(1) | Retrieves the value at the specified 2D coordinates. Does not perform bounds checking. |
| set(int x, int y, double value) | void | O(1) | Sets the value at the specified 2D coordinates. Does not perform bounds checking. |
| clear() | void | O(N) | Resets all values in the map to -1.0, where N is width * height. |
| index(int x, int y) | int | O(1) | Translates 2D coordinates into a 1D array index. |

## Integration Patterns

### Standard Usage
DoubleMap is intended to be used as a temporary data store within a single-threaded generation algorithm. The creator is responsible for all interactions.

```java
// A world generator creates a map for a specific task
int mapSize = 256;
DoubleMap temperatureMap = new DoubleMap(mapSize, mapSize);

// Populate the map using a noise function or other algorithm
for (int y = 0; y < mapSize; y++) {
    for (int x = 0; x < mapSize; x++) {
        double tempValue = NoiseGenerator.getTemperature(x, y);
        temperatureMap.set(x, y, tempValue);
    }
}

// Read a value for further processing
double centerTemp = temperatureMap.at(mapSize / 2, mapSize / 2);
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never share a single DoubleMap instance between multiple threads where at least one thread is writing to it, unless access is protected by an external lock. This is the most critical anti-pattern and will result in unpredictable behavior.

- **Out-of-Bounds Access:** The `at` and `set` methods do not validate input coordinates for performance reasons. The caller is responsible for ensuring all `x` and `y` values are within the `[0, width-1]` and `[0, height-1]` bounds, respectively. Failure to do so will result in an `ArrayIndexOutOfBoundsException`.

- **Long-Term Storage:** This class is not designed for serialization or long-term state persistence. It is a transient, in-memory structure. Data that must be saved should be converted to a more robust format.

## Data Pipeline
DoubleMap acts as an intermediate container in the world generation data pipeline. It holds the raw output of one stage, which then becomes the input for a subsequent stage.

> Flow:
> Procedural Noise Function (e.g., Simplex, Perlin) -> **DoubleMap** (stores raw height or climate values) -> Biome Generation Algorithm -> Final Chunk Block Data

