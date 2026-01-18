---
description: Architectural reference for Quad2d
---

# Quad2d

**Package:** com.hypixel.hytale.math.shape
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class Quad2d {
```

## Architecture & Concepts
The Quad2d class is a fundamental geometric primitive within the Hytale math library. It represents a quadrilateral in two-dimensional space, defined by four vertices: a, b, c, and d. This class is not a high-level engine system but rather a low-level, high-performance data structure used extensively for tasks such as:

*   **Collision Detection:** Defining the shape of 2D physics colliders.
*   **Rendering:** Specifying the boundaries of sprites, UI elements, or texture UV coordinates.
*   **Spatial Queries:** Representing areas of interest for culling, selection, or AI calculations.

Its design prioritizes performance and low memory allocation overhead over strict encapsulation. It achieves this by providing methods that can write results into pre-allocated Vector2d instances, a common and critical pattern in performance-sensitive game engine code.

## Lifecycle & Ownership
- **Creation:** A Quad2d is instantiated directly via its constructor (`new Quad2d(...)`) by any system that needs to represent a 2D quadrilateral. It is a lightweight object intended for frequent creation.
- **Scope:** The lifetime of a Quad2d instance is typically very short and local to the scope of the method that created it. It may exist for a single frame to represent a temporary state or calculation, or it may be a member of a longer-lived component, such as a UI widget or a physics body.
- **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as it is no longer referenced. There are no manual destruction or cleanup methods.

## Internal State & Concurrency
- **State:** **Mutable**. The internal Vector2d vertices are private, but the public getter methods (`getA`, `getB`, etc.) return direct references to these internal objects, not copies. Consequently, the state of a Quad2d can be modified externally by manipulating the Vector2d objects returned by its accessors. This is a deliberate performance trade-off.

- **Thread Safety:** **Not thread-safe**. This class contains no internal synchronization mechanisms. Concurrent access and modification from multiple threads will result in race conditions and undefined behavior. All operations on a Quad2d instance must be confined to a single thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Quad2d(a, b, c, d) | constructor | O(1) | Creates a quad from four Vector2d points. |
| getA() | Vector2d | O(1) | Returns a direct, mutable reference to the first vertex. |
| getMinX() / getMaxX() | double | O(1) | Calculates the minimum or maximum X coordinate across all vertices. |
| getMinY() / getMaxY() | double | O(1) | Calculates the minimum or maximum Y coordinate across all vertices. |
| getCenter() | Vector2d | O(1) | Calculates the center, allocating a new Vector2d for the result. |
| getCenter(target) | Vector2d | O(1) | Calculates the center and stores it in the provided `target` vector to prevent allocation. |
| getRandom(random, vec) | Vector2d | O(1) | Calculates a random point within the quad and stores it in the provided `vec` vector. |

## Integration Patterns

### Standard Usage
The most common and performant usage pattern involves creating a Quad2d and then using its calculation methods that accept a pre-allocated target vector. This avoids generating garbage and reduces pressure on the GC.

```java
// Create a quad representing a UI button's bounds
Vector2d[] corners = ...;
Quad2d buttonBounds = new Quad2d(corners);

// Pre-allocate a vector to store calculation results
Vector2d centerPoint = new Vector2d();

// Calculate the center without allocating a new vector
buttonBounds.getCenter(centerPoint);

// Now use centerPoint for positioning logic
```

### Anti-Patterns (Do NOT do this)
- **Unmanaged State Mutation:** Modifying a Quad2d's state via its getter methods is a significant source of bugs. This pattern breaks encapsulation and can lead to unpredictable behavior in systems that assume the quad is immutable for the duration of an operation.

    ```java
    // DANGEROUS: Modifying the internal state of the quad
    Quad2d sharedBounds = getSharedBounds();
    Vector2d vertexA = sharedBounds.getA();
    vertexA.set(100, 100); // This has now mutated sharedBounds for all other users
    ```

- **Multi-threaded Access:** Never share a Quad2d instance across threads without external locking. It is not designed for concurrent use.

## Data Pipeline
Quad2d is not a data processing component itself, but rather a data container that is passed through various engine pipelines.

> **Example Collision Pipeline:**
> Physics Body Definition -> **Quad2d** (as collider shape) -> Broad-phase Check -> Narrow-phase Collision Detection -> Collision Event

