---
description: Architectural reference for AreaScanner
---

# AreaScanner

**Package:** com.hypixel.hytale.builtin.hytalegenerator.scanners
**Type:** Transient

## Definition
```java
// Signature
public class AreaScanner extends Scanner {
```

## Architecture & Concepts

The AreaScanner is a composite component within the world generation framework. It does not perform point validation itself; instead, it implements the Decorator pattern to extend the behavior of another, subordinate **Scanner**. Its primary function is to transform a single-point scan operation into a multi-point area scan.

Upon instantiation, the AreaScanner pre-calculates a series of 2D column offsets within a specified **range** and **shape** (CIRCLE or SQUARE). Crucially, it sorts these offsets by their distance from the center.

When its `scan` method is invoked, it iterates through these pre-calculated offsets. For each offset, it creates a new execution context centered on that column and delegates the actual validation logic to its **childScanner**. This design allows for complex, layered scanning behaviors, such as using an AreaScanner to define a search region and a `ColumnScanner` to find valid ground-level positions within that region.

The outward-in scan order is a critical performance optimization. Combined with the **resultCap**, it allows the scanner to find the nearest valid positions and terminate its search early, avoiding an exhaustive and computationally expensive scan of the entire area.

## Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level world generation orchestrator, such as a ZoneGenerator or a specific feature placer. It is configured with a specific shape, range, and a child scanner instance tailored for a single, well-defined task.
-   **Scope:** The object's lifetime is ephemeral, strictly bound to the single generation operation for which it was created. It is not designed to be shared, reused, or persisted across different generation tasks.
-   **Destruction:** The AreaScanner holds no managed resources and requires no explicit cleanup. It becomes eligible for garbage collection as soon as the `scan` method returns and the caller releases its reference.

## Internal State & Concurrency
-   **State:** The AreaScanner is stateful but effectively immutable after construction. All configuration parameters, including the childScanner reference and the pre-computed `scanOrder` list, are finalized in the constructor and are not modified during its lifetime. The `scanOrder` list acts as an internal, read-only cache of scan positions.
-   **Thread Safety:** This class is thread-safe. The `scan` method operates exclusively on its immutable instance fields and method-local variables. It does not modify any shared state. However, the overall thread safety of an operation involving an AreaScanner is dependent on the thread safety of the **childScanner** it is configured with. The `Scanner.Context` object, which is passed down, is designed to carry thread-specific data, indicating the system's intent for concurrent execution.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| scan(Context context) | List<Vector3i> | O(N\*M) | Executes the scan. Iterates up to N columns in the area, where N is proportional to range^2. For each column, it invokes the child scanner, which has a complexity of M. Returns an empty list if resultCap is 0. |
| scanSpace() | SpaceSize | O(1) | Returns a clone of the pre-calculated bounding box that encompasses the entire potential scan area. |

## Integration Patterns

### Standard Usage
The AreaScanner must be composed with a child scanner to perform a useful operation. The typical pattern is to define a search area and then delegate column-specific logic to a more primitive scanner.

```java
// A hypothetical scanner that finds the first solid block in a column
Scanner columnScanner = new FindGroundScanner();

// Create an AreaScanner to search a 10-block radius circle for up to 5 valid ground positions
AreaScanner areaScanner = new AreaScanner(5, AreaScanner.ScanShape.CIRCLE, 10, columnScanner);

// The context provides the starting center point for the scan
Scanner.Context scanContext = new Scanner.Context(new Vector3i(100, 64, 250), ...);

// Execute the scan to get the results
List<Vector3i> groundPositions = areaScanner.scan(scanContext);
```

### Anti-Patterns (Do NOT do this)
-   **Unbounded Scans:** Configuring an AreaScanner with a large **range** and a high or unlimited **resultCap** can cause severe performance degradation. The system will be forced to invoke the child scanner for every single column in the vast area.
-   **Stateful Child Scanners:** Passing a non-thread-safe child scanner to an AreaScanner that will be used in a parallelized generator is a critical error. This will introduce race conditions and non-deterministic generation bugs. Child scanners should be immutable or use thread-local state.
-   **Incorrect Composition:** Nesting multiple AreaScanners can lead to an exponential explosion in the number of scan operations and is almost always a design error.

## Data Pipeline
The AreaScanner acts as a "fan-out, gather" component in the data pipeline. It transforms a single input context into many child contexts, executes them, and aggregates the results.

> Flow:
> Scanner.Context (Center Point) -> **AreaScanner** generates 2D offsets -> Creates new Scanner.Context for each offset -> Invokes **childScanner** -> **childScanner** returns valid 3D points -> **AreaScanner** aggregates points -> Final List<Vector3i>

