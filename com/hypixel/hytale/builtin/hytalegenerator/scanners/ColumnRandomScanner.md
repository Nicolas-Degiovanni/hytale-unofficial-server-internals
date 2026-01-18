---
description: Architectural reference for ColumnRandomScanner
---

# ColumnRandomScanner

**Package:** com.hypixel.hytale.builtin.hytalegenerator.scanners
**Type:** Transient

## Definition
```java
// Signature
public class ColumnRandomScanner extends Scanner {
```

## Architecture & Concepts
The ColumnRandomScanner is a specialized component within the world generation framework responsible for identifying a random subset of valid vertical positions within a single (X, Z) world column. Unlike exhaustive scanners that return all possible locations, this class is optimized for scenarios where only a few random, valid points are needed, such as placing ore veins, vegetation, or other sparse features.

Its primary architectural feature is the implementation of two distinct, performance-oriented search algorithms, defined by the **Strategy** enum. The choice of strategy is a critical design decision that directly impacts world generation performance based on the density of valid placement locations.

### Core Strategies

1.  **PICK_VALID:** This strategy is methodical and exhaustive. It first iterates through the entire specified Y-range, testing every single position against the provided Pattern. All valid positions are collected into a list. Finally, it randomly selects up to *resultsCap* positions from this list. This strategy guarantees a uniform random distribution among all possible valid locations but incurs a significant performance cost if the vertical search range is large.

2.  **DART_THROW:** This strategy is probabilistic and optimized for speed in large or sparse search spaces. It randomly selects, or "throws darts at," Y-coordinates within the search range and tests only those positions. It continues this process until it has found *resultsCap* valid positions or a fixed number of attempts has been exceeded. This avoids iterating the entire column, making it exceptionally fast for large Y-ranges, but it offers no guarantee of finding a position even if one exists.

The scanner's behavior is fully deterministic. It uses a SeedGenerator seeded by the absolute world coordinates of the scan operation, ensuring that world generation is perfectly reproducible.

## Lifecycle & Ownership
-   **Creation:** A ColumnRandomScanner is a transient, configured object. It is typically instantiated by a higher-level world generation orchestrator when parsing a feature's configuration. It is NOT a shared service or singleton.
-   **Scope:** The object's lifetime is tied to the specific world generation feature it was configured for. It is intended to be reused for all scan operations related to that feature within a world generation pass.
-   **Destruction:** The object holds no native resources and is managed by the Java garbage collector. Destruction is non-deterministic.

## Internal State & Concurrency
-   **State:** The internal state of a ColumnRandomScanner instance consists of its configuration parameters (minY, maxY, strategy, etc.). This state is **immutable**, as all fields are final and set only during construction.

-   **Thread Safety:** The class is **thread-safe** and its methods are re-entrant.
    -   The immutable nature of its internal state allows a single configured instance to be safely shared and called by multiple world generation worker threads simultaneously.
    -   The primary `scan` method does not modify any instance fields. All state required for the operation, such as the random number generator and result lists, is created on the stack within the method's local scope.
    -   Determinism across threads is guaranteed by the SeedGenerator, which produces a unique seed based on the (X, Y, Z) coordinates passed into the scan context.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| scan(Scanner.Context context) | List<Vector3i> | Variable | Executes the scan using the configured strategy. Complexity depends heavily on the strategy and Y-range. See Architecture & Concepts. |
| scanSpace() | SpaceSize | O(1) | Returns the geometric volume this scanner considers, which is always a 1x1 column. |

## Integration Patterns

### Standard Usage
The ColumnRandomScanner is intended to be configured once and then used by a generation system to find placement locations for a feature.

```java
// In a world generator, configure a scanner for finding diamond ore
ColumnRandomScanner.Strategy strategy = ColumnRandomScanner.Strategy.DART_THROW;
BiDouble2DoubleFunction bedrockFunction = null; // Or a function defining the world floor
int seed = 12345;

// Configure once
ColumnRandomScanner diamondScanner = new ColumnRandomScanner(5, 16, 8, seed, strategy, false, bedrockFunction);

// Reuse for each column being generated
for (int x = 0; x < 16; x++) {
    for (int z = 0; z < 16; z++) {
        Scanner.Context scanContext = createScannerContextFor(x, z);
        List<Vector3i> orePositions = diamondScanner.scan(scanContext);
        
        // Place ore at the returned positions
        placeOresAt(orePositions);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Strategy Mismatch:** Do not use the PICK_VALID strategy for extremely large vertical ranges (e.g., minY=0, maxY=256). The performance cost of iterating every block will severely impact generation speed. Use DART_THROW for such cases.
-   **Per-Call Instantiation:** Do not create a new ColumnRandomScanner for every call to `scan`. This is inefficient and defeats the purpose of having a reusable, configured component.
-   **Expecting Non-Determinism:** Do not use this scanner if you require true randomness. Its output is strictly deterministic based on the initial seed and the scan coordinates to ensure worlds are reproducible.

## Data Pipeline
The flow of data through the `scan` method is a direct transformation from a contextual request to a list of concrete world positions.

> Flow:
> Scanner.Context (Position, World Data, Pattern) -> **ColumnRandomScanner** -> Strategy Execution (PICK_VALID or DART_THROW) -> Pattern Matching -> Random Selection -> List<Vector3i> (World Positions)

