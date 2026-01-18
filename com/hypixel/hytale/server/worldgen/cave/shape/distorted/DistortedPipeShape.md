---
description: Architectural reference for DistortedPipeShape
---

# DistortedPipeShape

**Package:** com.hypixel.hytale.server.worldgen.cave.shape.distorted
**Type:** Transient

## Definition
```java
// Signature
public class DistortedPipeShape extends DistortedCylinderShape {
```

## Architecture & Concepts
The DistortedPipeShape is a mathematical primitive used by the server-side world generation system to define the volume of a cave segment. It specializes the more generic DistortedCylinderShape to model a continuous, pipe-like tunnel with ends that smoothly fade or taper into the surrounding environment.

Architecturally, this class serves as a data-driven model. It does not perform world modification itself; rather, it provides dimensional data to a separate voxel carving system. Its primary responsibility is to answer the question: "At a given point *t* along your central axis, what is your cross-sectional width and height?"

The key innovation over its parent class is the `compensation` factor and the associated logic in `getCompensatedDim`. This logic implements a fade-out effect for coordinates outside the normalized range of 0.0 to 1.0. This allows cave segments to blend seamlessly into larger caverns or terminate naturally without creating abrupt, flat walls. The shape is defined by start, middle, and end dimensions, which are interpolated using a provided function (e.g., cosine, linear) to create organic, non-uniform tunnels.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the nested `DistortedPipeShape.Factory`. This factory is a critical component, as it performs necessary pre-computation of compensation factors based on the pipe's orientation vector. Direct instantiation of DistortedPipeShape will result in incorrect and unpredictable geometry.
- **Scope:** The object's lifetime is exceptionally short. It is a transient data holder, instantiated by a cave network generator for the purpose of defining a single tunnel segment. It exists only long enough for a voxelization or carving process to query its dimensions.
- **Destruction:** The object is managed by the Java Garbage Collector. Once the world generation step that required it is complete, the instance is dereferenced and subsequently destroyed. No manual cleanup is necessary.

## Internal State & Concurrency
- **State:** The state of a DistortedPipeShape is **effectively immutable**. All defining parameters—origin, direction, dimensions, and compensation factor—are final and set during construction. The object serves as a read-only geometric definition.
- **Thread Safety:** This class is **thread-safe**. Due to its immutable nature, a single instance can be safely read by multiple worker threads simultaneously without locks or other synchronization primitives. This is a crucial design feature for enabling parallelized world generation, where multiple chunks may be carved concurrently.

## API Surface
The public contract is minimal, focusing entirely on querying the shape's dimensions at a normalized distance along its length.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWidthAt(double t) | double | O(1) | Returns the calculated cross-sectional width at normalized distance t. |
| getHeightAt(double t) | double | O(1) | Returns the calculated cross-sectional height at normalized distance t. |
| isValidProjection(double t) | boolean | O(1) | Always returns true. Indicates the shape is considered valid for projection tests at any distance. |

## Integration Patterns

### Standard Usage
The only correct way to create and use a DistortedPipeShape is through its factory, which is typically provided to a higher-level generation algorithm.

```java
// A world generator receives a factory for the desired shape
DistortedShape.Factory shapeFactory = new DistortedPipeShape.Factory();

// The generator defines the parameters for a new cave segment
Vector3d origin = new Vector3d(100, 50, 200);
Vector3d direction = new Vector3d(0.5, 0.1, 0.8).normalize();
double length = 50.0;

// The factory creates the shape, performing necessary pre-computation
DistortedShape pipe = shapeFactory.create(
    origin,
    direction,
    length,
    /* startW/H */ 8.0, 8.0,
    /* midW/H */ 12.0, 10.0,
    /* endW/H */ 7.0, 7.0,
    GeneralNoise.InterpolationFunction.COSINE
);

// The pipe instance is then passed to a voxel carver system.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new DistortedPipeShape(...)` directly. The constructor is public but bypasses the critical calculations performed by the `DistortedPipeShape.Factory`, such as `getCompensationFactor` and `getHeightCompensation`. Doing so will lead to geometrically incorrect shapes that do not align with the world's vertical axis properly.
- **State Modification:** Do not attempt to modify the object's state via reflection. The world generation system relies on the immutability of these shapes for thread safety and predictable outcomes.

## Data Pipeline
DistortedPipeShape acts as a data source in the middle of the cave generation pipeline. It translates high-level parameters into a concrete, queryable geometric form.

> Flow:
> Cave Network Algorithm -> Defines segment parameters (origin, direction, length) -> **DistortedPipeShape.Factory** -> Creates **DistortedPipeShape** instance -> Voxel Carver System -> Queries shape dimensions -> Modifies World Voxel Grid

---

