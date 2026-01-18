---
description: Architectural reference for CoordinateOriginRotator
---

# CoordinateOriginRotator

**Package:** com.hypixel.hytale.procedurallib.random
**Type:** Transient

## Definition
```java
// Signature
public class CoordinateOriginRotator extends CoordinateRotator {
```

## Architecture & Concepts
The CoordinateOriginRotator is a specialized transformation utility within the procedural generation library. It extends the functionality of the base CoordinateRotator to perform rotations around an arbitrary point in 3D space, rather than being fixed to the world origin (0,0,0).

This class implements a standard geometric pattern for off-origin rotation:
1.  **Translate:** The input coordinate is first translated so that the specified origin point becomes the new system origin (0,0,0).
2.  **Rotate:** The translated coordinate is then rotated using the parent CoordinateRotator's logic.
3.  **Translate Back:** The rotated coordinate is translated back by the original origin offset.

This component is crucial for procedural algorithms that need to apply localized rotations to features. For example, rotating a procedurally generated structure or a cluster of foliage around its own center point without affecting its global world position.

**Warning:** The method names, such as randomDoubleX, are inherited from a broader abstraction related to noise sampling. The operations performed by this class are entirely deterministic geometric rotations, not random processes. The `seed` parameter is ignored.

## Lifecycle & Ownership
-   **Creation:** Instantiated directly by procedural generation algorithms when a localized rotation is required. It is not managed by a central service or dependency injection framework.
-   **Scope:** Short-lived and task-specific. An instance typically exists only for the duration of a single, focused generation task, such as placing objects within a single world chunk.
-   **Destruction:** The object is managed by the Java Garbage Collector. Once the generation algorithm that created it completes and releases its reference, the instance becomes eligible for collection. No explicit cleanup is necessary.

## Internal State & Concurrency
-   **State:** Immutable. The origin coordinates (`originX`, `originY`, `originZ`) and the inherited rotation angles (`pitch`, `yaw`) are final fields set exclusively during construction. The object's state cannot be modified after creation.
-   **Thread Safety:** This class is inherently thread-safe. Due to its immutable nature, a single instance can be safely shared and accessed by multiple threads simultaneously without the need for external locking or synchronization.

## API Surface
The public API consists of overrides that apply the origin-centric rotation logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| randomDoubleX(seed, x, y) | double | O(1) | Rotates a 2D point around the configured origin and returns the new X coordinate. |
| randomDoubleY(seed, x, y) | double | O(1) | Rotates a 2D point around the configured origin and returns the new Y coordinate. |
| randomDoubleX(seed, x, y, z) | double | O(1) | Rotates a 3D point around the configured origin and returns the new X coordinate. |
| randomDoubleY(seed, x, y, z) | double | O(1) | Rotates a 3D point around the configured origin and returns the new Y coordinate. |
| randomDoubleZ(seed, x, y, z) | double | O(1) | Rotates a 3D point around the configured origin and returns the new Z coordinate. |

## Integration Patterns

### Standard Usage
Create an instance once with the desired rotation and origin, then reuse it to transform multiple points within that coordinate system.

```java
// Define a 45-degree yaw rotation around a central point (1000, 64, 2500)
CoordinateRotator rotator = new CoordinateOriginRotator(0.0, 45.0, 1000.0, 64.0, 2500.0);

// A point relative to the origin we want to rotate
double worldX = 1010.0;
double worldY = 64.0;
double worldZ = 2500.0;

// Calculate the new position of the point after rotation
double newX = rotator.randomDoubleX(0, worldX, worldY, worldZ);
double newZ = rotator.randomDoubleZ(0, worldX, worldY, worldZ);

// newX will be approximately 1007.07
// newZ will be approximately 2507.07
```

### Anti-Patterns (Do NOT do this)
-   **Instantiation in a Loop:** Avoid creating a new CoordinateOriginRotator inside a tight loop that processes many coordinates. This is highly inefficient and generates unnecessary garbage. Instantiate it once before the loop.
-   **Relying on the Seed:** Do not expect the `seed` parameter to have any effect on the output. The rotation is deterministic. Passing different seeds will yield identical results.

## Data Pipeline
The class acts as a pure transformation stage in a data pipeline. It accepts a coordinate, applies a series of mathematical operations, and outputs a new coordinate.

> Flow:
> World Coordinate (x,y,z) -> **CoordinateOriginRotator** (Translate -> Delegate Rotation -> Translate Back) -> Rotated World Coordinate (x',y',z')

