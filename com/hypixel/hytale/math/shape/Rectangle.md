---
description: Architectural reference for Rectangle
---

# Rectangle

**Package:** com.hypixel.hytale.math.shape
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class Rectangle {
```

## Architecture & Concepts
This class is a fundamental geometric primitive representing an axis-aligned bounding box (AABB) in 2D space. It is a core data structure used throughout the engine for spatial calculations, including collision detection, culling, and user interface layout.

Unlike service-oriented classes, Rectangle is a simple data container. Its primary architectural role is to provide a common, mutable representation of a 2D area, defined by its minimum (bottom-left) and maximum (top-right) corner points. Its mutability is a key design choice, favoring performance by allowing in-place modification to reduce object allocation during intensive calculations, such as in the physics or rendering loops.

### Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructors (`new Rectangle(...)`) by any system requiring a 2D spatial representation. It is not managed by any dependency injection framework or service locator.
- **Scope:** Typically method-scoped or a member field of a larger component, for example, a UI widget's bounds. Its lifetime is tied directly to its owner.
- **Destruction:** The object is eligible for garbage collection as soon as it is no longer referenced. No explicit cleanup or `destroy` method is required.

## Internal State & Concurrency
- **State:** The state is mutable and is composed of two `Vector2d` objects, `min` and `max`. These vectors are also mutable, and the class provides direct access to them.
- **Thread Safety:** **This class is not thread-safe.** Its internal state can be modified at any time via the `assign` method or by directly mutating the `Vector2d` objects returned by `getMin` and `getMax`. Concurrent access from multiple threads without external synchronization will lead to race conditions and unpredictable behavior.

**WARNING:** Sharing instances of Rectangle between threads is highly discouraged. If required, all access must be protected by explicit locking mechanisms.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Rectangle() | constructor | O(1) | Creates a zero-sized rectangle at the origin. |
| Rectangle(min, max) | constructor | O(1) | Creates a rectangle from two corner vectors. |
| getMin() | Vector2d | O(1) | Returns a direct reference to the internal min vector. |
| getMax() | Vector2d | O(1) | Returns a direct reference to the internal max vector. |
| assign(...) | Rectangle | O(1) | Modifies the rectangle's bounds in-place. Returns self for chaining. |
| hasArea() | boolean | O(1) | Checks if the rectangle has a positive, non-zero area. |

## Integration Patterns

### Standard Usage
Rectangle is intended for use as a temporary or member object for spatial calculations.
```java
// Example: Checking if a point is inside a UI element's bounds
Rectangle uiBounds = new Rectangle(10, 10, 100, 50);
Vector2d cursorPosition = new Vector2d(25, 25);

if (uiBounds.hasArea() && cursorPosition.x >= uiBounds.getMinX() && cursorPosition.x < uiBounds.getMaxX()) {
    // Point is within the horizontal bounds
}
```

### Anti-Patterns (Do NOT do this)
- **Encapsulation Breach:** Do not modify the internal state of a Rectangle by mutating the vectors returned by `getMin` or `getMax`. This creates non-obvious side effects and breaks the class contract.
    ```java
    // BAD: Unpredictable side effects
    Rectangle bounds = getSomeSharedBounds();
    Vector2d min = bounds.getMin();
    min.x = -1000; // This modifies the shared Rectangle instance
    ```
- **Concurrent Modification:** Do not read a Rectangle from one thread while another thread might be calling `assign` on it. This will result in reading partially updated, inconsistent state.
- **Assuming Immutability:** Do not treat this class as an immutable value type. Passing a Rectangle to a method gives that method the ability to change its state. If immutability is required, create a defensive copy using the copy constructor.
    ```java
    // GOOD: Defensive copy
    public void processBounds(Rectangle originalBounds) {
        Rectangle localCopy = new Rectangle(originalBounds);
        // ... perform work on localCopy
    }
    ```

## Data Pipeline
Rectangle is not a processing stage in a pipeline; it is the *data* that flows through various engine systems. It acts as a data carrier for spatial information.

> **Example Flow (UI Hit-Testing):**
> Input System (produces cursor `Vector2d`) -> UI Manager -> **Rectangle** (defines widget bounds) -> Hit-Test Logic -> Event Dispatch

In this flow, the Rectangle object provides the essential boundary data required by the "Hit-Test Logic" to determine if an input event applies to a specific UI widget.

