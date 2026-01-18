---
description: Architectural reference for InterpolatedCurve
---

# InterpolatedCurve

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.math
**Type:** Transient

## Definition
```java
// Signature
public class InterpolatedCurve implements Double2DoubleFunction {
```

## Architecture & Concepts
The **InterpolatedCurve** is a fundamental mathematical utility within the world generation framework. It acts as a composable function blender, designed to smoothly transition between two distinct **Double2DoubleFunction** implementations over a specified domain.

Architecturally, it serves as a powerful node in a procedural generation graph. By implementing the standard **Double2DoubleFunction** interface, instances of **InterpolatedCurve** can be chained together or nested, allowing for the creation of complex, multi-stage transitions. For example, it can be used to blend a flat plains noise function into a mountainous noise function, creating a natural-looking foothills region.

The core of its behavior is a cosine-based interpolation, which provides a visually pleasing ease-in/ease-out effect, controlled by the **smoothTransition** parameter. A value of 0.0 results in a linear blend, while a value of 1.0 results in a full S-curve transition.

## Lifecycle & Ownership
- **Creation:** Instances are created directly via the public constructor: `new InterpolatedCurve(...)`. It is not managed by a service locator or dependency injection framework. It is typically instantiated by higher-level generator logic that needs to combine two functional profiles.
- **Scope:** The object is transient and its lifetime is bound to the scope of the generator component that created it. It holds no external resources and is designed to be a lightweight value object.
- **Destruction:** The object is managed by the Java Garbage Collector. No explicit cleanup or destruction methods are required.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields are final and are assigned exclusively during construction. Once an **InterpolatedCurve** is created, its blending behavior is fixed for its entire lifetime.
- **Thread Safety:** **Inherently Thread-Safe**. Due to its immutable nature, a single **InterpolatedCurve** instance can be safely shared and accessed by multiple threads simultaneously without any external synchronization. This is a critical feature for parallelized world generation tasks.

    **WARNING:** While the **InterpolatedCurve** class itself is thread-safe, the overall thread safety depends on the provided **functionA** and **functionB** implementations. The caller is responsible for ensuring that the functions passed to the constructor are also thread-safe if the curve is to be used in a concurrent context.

## API Surface
The public contract is focused on its role as a function.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| InterpolatedCurve(posA, posB, smooth, funcA, funcB) | constructor | O(1) | Constructs a new curve to blend between funcA and funcB. Throws IllegalArgumentException if smoothTransition is not in the [0, 1] range. |
| get(double x) | double | O(1) | Returns the computed value at point x. The complexity assumes the underlying functions funcA and funcB are also O(1). |
| transitionCurve(double ratio) | double | O(1) | A public helper that transforms a linear ratio [0, 1] into the smoothed transition value. |

## Integration Patterns

### Standard Usage
The primary use case is to define a transition zone between two mathematical functions, such as noise generators for different biomes.

```java
// Example: Blend from a flat terrain (y=10) to a hilly terrain (sine wave)
// between x=100 and x=200.

Double2DoubleFunction plains = (x) -> 10.0;
Double2DoubleFunction hills = (x) -> 10.0 + Math.sin(x / 20.0) * 5.0;

// Create a curve that smoothly transitions between the two functions.
double transitionStart = 100.0;
double transitionEnd = 200.0;
double smoothness = 0.75; // A mostly smooth S-curve transition

InterpolatedCurve terrainFunction = new InterpolatedCurve(
    transitionStart,
    transitionEnd,
    smoothness,
    plains,
    hills
);

// Now use the combined function to get terrain height.
double heightAt90 = terrainFunction.get(90.0);   // Result from 'plains'
double heightAt150 = terrainFunction.get(150.0); // Blended result
double heightAt210 = terrainFunction.get(210.0); // Result from 'hills'
```

### Anti-Patterns (Do NOT do this)
- **Re-instantiation in a Loop:** Do not create a new **InterpolatedCurve** for every value you need to calculate. The object is lightweight but should be instantiated once and reused for all calculations within its intended domain.
- **Ignoring Thread Safety of Inputs:** Passing stateful or non-thread-safe lambda expressions or function objects into the constructor for use in a multi-threaded generator will lead to race conditions and unpredictable results.
- **Overlapping Transition Zones:** While technically possible, nesting **InterpolatedCurve** instances with poorly defined or overlapping transition zones (**positionA**, **positionB**) can create complex and difficult-to-debug generation artifacts.

## Data Pipeline
**InterpolatedCurve** acts as a transformation stage in a data pipeline, typically processing spatial coordinates.

> Flow:
> Input Coordinate (double) -> **InterpolatedCurve.get()** -> Evaluates underlying functions -> Blends results -> Output Value (double)

