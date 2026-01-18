---
description: Architectural reference for ColumnLinearScanner
---

# ColumnLinearScanner

**Package:** com.hypixel.hytale.builtin.hytalegenerator.scanners
**Type:** Transient

## Definition
```java
// Signature
public class ColumnLinearScanner extends Scanner {
```

## Architecture & Concepts
The ColumnLinearScanner is a specialized strategy object within the Hytale World Generator framework. Its primary function is to identify valid placement locations for game features (such as vegetation, ores, or structures) by performing a search along a single vertical column (Y-axis) at a given (X, Z) coordinate.

This class embodies the Strategy Pattern, where different `Scanner` implementations provide distinct methods for searching the world space. The ColumnLinearScanner implements the simplest of these: a linear, one-dimensional search. It is configured with vertical boundaries and a search direction, and then invoked to test a provided `Pattern` against each block in the column.

Its behavior is heavily configurable through its constructor, allowing world designers to define precise vertical ranges, anchor these ranges relative to the world or a dynamic heightmap, and limit the number of results. This makes it a fundamental and highly reusable building block for procedural generation tasks that require placement based on vertical context, such as placing trees on top of terrain or ores below a certain depth.

### Lifecycle & Ownership
- **Creation:** Instantiated directly by higher-level world generation components, such as a `FeaturePlacer` or `Decorator`. It is not managed by a dependency injection container or a central registry. Each instance is configured for a specific generation task.
- **Scope:** Extremely short-lived. An instance is typically created, its `scan` method is called once, and it is then immediately eligible for garbage collection. It holds no state beyond its initial configuration.
- **Destruction:** Handled by the standard Java garbage collector. No explicit cleanup methods are required.

## Internal State & Concurrency
- **State:** The internal state of a ColumnLinearScanner instance is **immutable**. All configuration fields (`minY`, `maxY`, `resultsCap`, etc.) are final and set only during construction. The class itself does not cache results or modify its own state during the `scan` operation.
- **Thread Safety:** The class is inherently thread-safe and re-entrant due to its immutable state. The `scan` method is safe to be called from multiple worker threads simultaneously, provided that each call receives a unique or thread-local `Scanner.Context` object. The method's internal operations use local variables (`patternPosition`, `validPositions`) that do not create side effects or race conditions between concurrent executions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| scan(Context context) | List<Vector3i> | O(N) | Executes the vertical scan. N is the height of the scan range (maxY - minY). Throws NullPointerException if context is null. |
| scanSpace() | SpaceSize | O(1) | Returns the theoretical maximum volume this scanner might inspect, which for this class is a 1x1 column. |

## Integration Patterns

### Standard Usage
The ColumnLinearScanner is designed to be instantiated and used immediately by a world generation process that needs to find a valid Y-coordinate for a given (X, Z) location.

```java
// A world generator component receives a context for a specific location.
Scanner.Context scanContext = ...; // Provided by the generator framework

// Configure and create a scanner to find the first valid position from the top down,
// within 20 blocks of the current position.
ColumnLinearScanner scanner = new ColumnLinearScanner(-10, 10, 1, true, true, null);

// Execute the scan to find potential placement spots.
List<Vector3i> validPlacements = scanner.scan(scanContext);

if (!validPlacements.isEmpty()) {
    // Place the feature at validPlacements.get(0)
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not attempt to modify a ColumnLinearScanner after creation or reuse a single instance for different placement rules. They are cheap to create and should be instantiated with the specific configuration needed for each distinct task.
- **Ignoring Context:** The `scan` method's behavior is critically dependent on the provided `Scanner.Context`. Passing a stale or incorrect context will lead to features being placed in invalid locations or not at all.
- **Misunderstanding Relative vs Absolute Coordinates:** The `isRelativeToPosition` and `baseHeightFunction` parameters fundamentally change how the vertical scan range is calculated. Misconfiguring these is a common source of generation bugs, causing features to float in the air or be buried deep underground.

## Data Pipeline
The ColumnLinearScanner processes data by transforming a starting coordinate and a set of rules into a list of valid world positions.

> Flow:
> World Generator provides `Scanner.Context` (includes start position, world data access, and a `Pattern`) -> **ColumnLinearScanner** calculates vertical scan bounds (minY, maxY) based on its configuration -> It iterates from top-down or bottom-up -> For each Y-level, it invokes `Pattern.matches()` -> If match is true, the position is added to a results list -> The final list of valid `Vector3i` positions is returned.

