---
description: Architectural reference for GapPattern
---

# GapPattern

**Package:** com.hypixel.hytale.builtin.hytalegenerator.patterns
**Type:** Transient

## Definition
```java
// Signature
public class GapPattern extends Pattern {
```

## Architecture & Concepts

The GapPattern is a high-level, composite pattern used within the procedural world generation system. It defines a complex spatial rule characterized by a central "gap" region and surrounding "anchor" structures, which are then extruded along specified angles and vertical depths.

Architecturally, its most critical feature is its use of **pre-computation**, also known as baking. Upon instantiation, the GapPattern translates its declarative configuration—such as angles, sizes, and depths—into a concrete, cached set of discrete spatial points. Each point is paired with a subordinate Pattern (either a gap or an anchor).

This design pattern is chosen for performance. The expensive trigonometric and vector calculations required to lay out the pattern in 3D space are performed only once, during construction. Subsequent calls to the core matching logic are reduced to highly efficient iterations over these pre-computed lists. This allows a single, complex GapPattern instance to be evaluated against thousands of world coordinates with minimal computational overhead per check.

It functions as a stateful but immutable configuration object that encapsulates a specific geometric test.

## Lifecycle & Ownership

-   **Creation:** A GapPattern is instantiated by a higher-level generation orchestrator, typically when processing a biome or zone configuration file. It is not created on-the-fly within tight generation loops.
-   **Scope:** The object's lifetime is tied to the generation phase for which its rule is relevant. It is designed to be created once and reused for all matching operations within that phase.
-   **Destruction:** The instance becomes eligible for garbage collection as soon as the world generator that created it completes its task or moves to a new phase that does not require this specific pattern.

## Internal State & Concurrency

-   **State:** The internal state is **effectively immutable** post-construction. All internal lists of positioned patterns and the overall bounding box (readSpaceSize) are populated exclusively within the constructor. The object acts as a read-only cache for the results of its initial geometric computations.
-   **Thread Safety:** This class is **thread-safe for all read operations**. The primary public method, matches, only reads from the pre-computed internal state and involves no mutation. This is a deliberate design choice, enabling a single GapPattern instance to be safely and concurrently evaluated by multiple world generation worker threads without requiring any locks.

    **WARNING:** The constructor is not thread-safe and performs mutations on its internal state. An instance must be fully constructed by a single thread before being shared with other threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(Context context) | boolean | O(N\*M) | Evaluates the pattern against a given world context. Returns true if the context satisfies the pattern's rules. N is the number of configured angles, and M is the average number of points per direction. |
| readSpace() | SpaceSize | O(1) | Returns a clone of the pre-calculated bounding box that fully contains the pattern. This is a low-cost operation. |

## Integration Patterns

### Standard Usage

The GapPattern should be instantiated once with its full configuration and then reused for multiple checks within a generation pass. The `readSpace` method can be used by orchestrators as a coarse filter to quickly discard checks that are far outside the pattern's area of influence.

```java
// A generator creates the pattern from configuration
GapPattern caveOpeningPattern = new GapPattern(angles, gapSize, ...);

// Later, in a tight loop across many world coordinates
for (int x = 0; x < 100; x++) {
    Pattern.Context checkContext = new Pattern.Context(world, new Vector3i(x, y, z));
    
    // The expensive pattern is reused for many cheap checks
    if (caveOpeningPattern.matches(checkContext)) {
        // Place cave entrance blocks
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Re-instantiation in a Loop:** Never create a new GapPattern inside a loop that checks world coordinates. The constructor is computationally expensive and is designed to be called once per configuration. Doing so will severely degrade generator performance.

    ```java
    // BAD: Constructor is called repeatedly, causing massive performance loss.
    for (int x = 0; x < 100; x++) {
        GapPattern badPattern = new GapPattern(angles, gapSize, ...); // DO NOT DO THIS
        if (badPattern.matches(context)) { ... }
    }
    ```

-   **Ignoring readSpace:** Failing to use the `readSpace` bounding box for broad-phase culling can lead to unnecessary calls to the `matches` method. While `matches` is fast, avoiding the call entirely is faster.

## Data Pipeline

The flow of data for a pattern check is simple and direct. The "data" is the spatial context provided by the world generator.

> Flow:
> World Generator provides `Pattern.Context` -> **GapPattern.matches(context)** -> Iterates internal pre-computed `PositionedPattern` lists -> Boolean result -> Generator makes voxel placement decision
---

