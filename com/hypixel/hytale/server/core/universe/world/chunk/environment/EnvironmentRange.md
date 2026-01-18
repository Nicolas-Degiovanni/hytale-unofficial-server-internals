---
description: Architectural reference for EnvironmentRange
---

# EnvironmentRange

**Package:** com.hypixel.hytale.server.core.universe.world.chunk.environment
**Type:** Transient

## Definition
```java
// Signature
public class EnvironmentRange {
```

## Architecture & Concepts
The EnvironmentRange is a fundamental data structure, not a service. It acts as a value object that represents a discrete, vertical segment of a world column. Its primary role is to associate a specific environment, identified by an integer ID, with a contiguous range of Y-coordinates (from a minimum to a maximum height).

Within the server's world generation system, a collection of EnvironmentRange objects is used to define the complete environmental profile of a single (X, Z) world column. For example, one column might be composed of multiple ranges: one for a deep cave system, another for a stone layer, and a final one for the surface biome.

This class is a foundational building block for procedural generation, enabling algorithms to reason about and manipulate environmental strata before committing to final block placement.

### Lifecycle & Ownership
-   **Creation:** Instances are created dynamically by higher-level world generation or chunk population systems. They are typically instantiated when an algorithm partitions a vertical column into different environmental zones.
-   **Scope:** The lifetime of an EnvironmentRange is strictly tied to its parent container, which is usually a chunk or a specialized data structure representing a world column. It persists in memory only as long as the corresponding chunk is loaded.
-   **Destruction:** The object is eligible for garbage collection once its parent chunk is unloaded and no other references to it exist. It has no explicit destruction or cleanup logic.

## Internal State & Concurrency
-   **State:** EnvironmentRange is a mutable data container. The package-private visibility of its setters (setMin, setMax, setId) is a critical design choice. It signals that mutations are expected, but are intended to be controlled by closely-related classes within the same package, such as an EnvironmentColumn manager that might merge, split, or adjust ranges during generation.
-   **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization mechanisms. All access and modification must be externally synchronized or confined to a single thread. It is assumed that the world generation pipeline that owns these objects is responsible for managing concurrency at a higher level.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EnvironmentRange(id) | constructor | O(1) | Creates a range spanning nearly the entire world height with the given ID. |
| EnvironmentRange(min, max, id) | constructor | O(1) | Creates a range for a specific vertical segment and ID. |
| getMin() | int | O(1) | Returns the inclusive minimum Y-coordinate of the range. |
| getMax() | int | O(1) | Returns the inclusive maximum Y-coordinate of the range. |
| getId() | int | O(1) | Returns the identifier for the environment this range represents. |
| height() | int | O(1) | Calculates the total vertical size of the range. |
| copy() | EnvironmentRange | O(1) | Creates a new, deep copy of the instance. Essential for safe, state-detached operations. |

## Integration Patterns

### Standard Usage
EnvironmentRange objects are typically created and managed in collections by a world generator or a chunk environment manager. They are not retrieved from a service locator or context.

```java
// A generator defining the environmental layers for a single world column.
// The IDs would typically be constants representing specific environments.
List<EnvironmentRange> environmentLayers = new ArrayList<>();

// Bedrock Layer
environmentLayers.add(new EnvironmentRange(0, 4, ENV_BEDROCK));

// Deepslate Cave Layer
environmentLayers.add(new EnvironmentRange(5, 32, ENV_DEEPSLATE_CAVES));

// Surface Forest Layer
environmentLayers.add(new EnvironmentRange(64, 80, ENV_FOREST_SURFACE));
```

### Anti-Patterns (Do NOT do this)
-   **Shared Mutable References:** Do not pass an EnvironmentRange instance to multiple independent systems that may modify it. Its mutable nature can lead to unpredictable behavior and data corruption. If a detached copy is needed, always use the **copy()** method.
-   **Multi-threaded Modification:** Never modify an EnvironmentRange instance from multiple threads without external locking. The object is not designed for concurrent access and will result in race conditions.

## Data Pipeline
EnvironmentRange is not a processing node in a pipeline; rather, it is the data *payload* that flows through the world generation pipeline.

> Flow:
> Procedural Generation Algorithm -> **Creates EnvironmentRange instances** -> Stored in an in-memory Chunk data structure -> Read by Block Placement Logic -> Serialized to region file on disk

