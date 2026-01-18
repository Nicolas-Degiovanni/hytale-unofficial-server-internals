---
description: Architectural reference for BorderDistanceFunction
---

# BorderDistanceFunction

**Package:** com.hypixel.hytale.procedurallib.logic.cell
**Type:** Transient

## Definition
```java
// Signature
public class BorderDistanceFunction implements CellDistanceFunction {
```

## Architecture & Concepts

The BorderDistanceFunction is a specialized implementation of the CellDistanceFunction interface that operates as a **Decorator**. It wraps an existing CellDistanceFunction to fundamentally alter its output, transforming a standard cellular noise pattern into one with distinct, hard borders.

Its primary role within the procedural generation engine is to create sharp transitions between cellular regions. Instead of calculating the distance to the center of a cell (as in traditional Voronoi noise), this function first identifies the containing cell and then, if a density condition is met, calculates the distance to the *closest point on that cell's border*.

This is achieved by composing several components:
1.  A base **CellDistanceFunction**: The underlying cellular pattern to be modified.
2.  A **PointEvaluator** for borders: Defines the shape and jitter of the points that constitute the "border".
3.  A **density condition**: A predicate that determines if a cell is "active". If a cell is not active, it is effectively ignored, creating empty space.

This composite approach allows for the generation of complex, non-uniform patterns like cracked earth, distinct biome regions, or city blocks from a simple cellular foundation. It acts as a powerful modifier in a procedural generation pipeline.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly via its public constructor, typically during the setup phase of a world generator or a specific procedural algorithm. It is a building block, composed with other functions to achieve a desired effect.
-   **Scope:** The object's lifetime is bound to the parent generator that created it. It is not a global singleton and persists only for the duration of a specific generation task.
-   **Destruction:** The BorderDistanceFunction is eligible for garbage collection once the procedural generation task is complete and all references to the parent generator are released.

## Internal State & Concurrency
-   **State:** The class is stateful but **effectively immutable** after construction. Its internal fields, which reference the wrapped distance function and evaluators, are assigned once in the constructor and never modified. All calculations are performed on method-local variables or by writing to a supplied ResultBuffer.
-   **Thread Safety:** This class is **thread-safe**. Its methods are pure functions that depend only on their arguments. The use of a mutable ResultBuffer passed into each method call is a key design choice that prevents internal state mutation and allows concurrent execution across multiple threads without locks or race conditions, assuming the composed dependencies (distanceFunction, evaluators) are also thread-safe.

## API Surface

The public contract is dominated by the implementation of the CellDistanceFunction interface. The core logic resides within the `transition2D` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BorderDistanceFunction(distanceFunction, borderEvaluator, density) | constructor | O(1) | Constructs the decorator with its required dependencies. |
| transition2D(seed, x, y, cellX, cellY, buffer, pointEvaluator) | void | O(1) | Core operational method. Calculates the nearest border point for a given coordinate. Complexity is constant relative to the number of neighbor cells searched (typically 9 or 25). |
| nearest2D(...) | void | O(1) | Alias for transition2D. |
| scale(value) | double | O(1) | Delegates the scaling operation to the wrapped distance function. |
| invScale(value) | double | O(1) | Delegates the inverse scaling operation to the wrapped distance function. |
| getCellX(x, y) | int | O(1) | Delegates cell index calculation to the wrapped distance function. |
| getCellY(x, y) | int | O(1) | Delegates cell index calculation to the wrapped distance function. |

**Warning:** All 3D-related methods (`nearest3D`, `transition3D`, etc.) are not implemented and will throw an `UnsupportedOperationException` at runtime. This component is strictly for 2D generation.

## Integration Patterns

### Standard Usage

The BorderDistanceFunction is designed to be composed with other procedural components. It wraps a base distance function to add the border-finding logic.

```java
// 1. Define a base cellular distance function
CellDistanceFunction baseFunction = new ChebyshevDistanceFunction();

// 2. Define how border points are generated
PointEvaluator borderEvaluator = new JitterPointEvaluator(NormalPointEvaluator.EUCLIDEAN, new Jitter(0.5, 0.5, 0.0));

// 3. Define a condition for which cells are active (e.g., 50% chance)
IDoubleCondition densityCondition = (hash) -> (hash & 1) == 0;

// 4. Create the BorderDistanceFunction by wrapping the base function
CellDistanceFunction borderedFunction = new BorderDistanceFunction(baseFunction, borderEvaluator, densityCondition);

// 5. Use the composed function in a generation algorithm
// The result will be a cellular pattern with hard edges instead of smooth gradients.
generator.setDistanceFunction(borderedFunction);
```

### Anti-Patterns (Do NOT do this)
-   **Using in 3D Contexts:** Do not attempt to use this class for 3D procedural generation. The 3D methods are not implemented and will cause a runtime crash.
-   **Misconfigured Jitter:** Providing a `borderEvaluator` with a jitter value greater than 0.5 can cause the search area for borders to expand significantly, leading to performance degradation and potentially incorrect results as border searches bleed into non-adjacent cells.
-   **Null Dependencies:** Passing null for the `distanceFunction`, `borderEvaluator`, or `density` will result in a `NullPointerException`. The class relies on these components to function.

## Data Pipeline

The flow of data for a single 2D coordinate query demonstrates the decorating and conditional logic of the class.

> Flow:
> (x, y) Coordinate -> **BorderDistanceFunction.transition2D**
> 1.  Delegates to wrapped `distanceFunction.nearest2D` to find the primary cell center and its hash.
> 2.  The primary cell hash is evaluated by the `density` condition.
> 3.  **If density check fails:** The result distance is set to 0.0 and the process terminates.
> 4.  **If density check passes:** A 3x3 or 5x5 grid of cells around the primary cell is scanned.
> 5.  For each cell in the grid, `distanceFunction.evalPoint2` is called with the `borderEvaluator`.
> 6.  The minimum distance found during the grid scan is identified.
> 7.  Final distance and point data are written into the provided `ResultBuffer`.
> -> Populated `ResultBuffer`

