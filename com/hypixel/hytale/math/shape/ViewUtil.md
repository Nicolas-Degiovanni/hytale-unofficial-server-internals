---
description: Architectural reference for ViewUtil
---

# ViewUtil

**Package:** com.hypixel.hytale.math.shape
**Type:** Utility

## Definition
```java
// Signature
public class ViewUtil {
```

## Architecture & Concepts
ViewUtil is a low-level, stateless mathematical utility class designed for 2D geometric clipping. It provides a specific implementation of the Cohen-Sutherland line clipping algorithm, a highly efficient method for determining which parts of a line segment are visible within a rectangular clipping area.

The core design assumes a normalized device coordinate (NDC) space, where the visible area is a 2x2 square centered at the origin, with coordinates ranging from -1.0 to 1.0 on both axes. The class uses bitwise flags to classify the position of a point relative to this clipping rectangle, enabling rapid acceptance of fully visible lines and rejection of fully invisible lines.

This component serves as a foundational building block for higher-level rendering or culling systems. Its purpose is to perform fast, preliminary visibility tests to avoid passing unnecessary geometry to more computationally expensive stages of a rendering pipeline.

**Warning:** The primary algorithm method, CohenSutherlandLineClipAndDraw, is misleadingly named. It performs the clipping calculation but does **not** execute any drawing operations, despite accepting a Graphics2D context as a parameter. Furthermore, all computational methods in this class are declared as private, rendering it unusable from outside its own source file. This suggests it is either an internal helper for a sibling class or an incomplete component.

## Lifecycle & Ownership
- **Creation:** This class is never instantiated. It enforces a strict utility pattern by providing a private constructor that throws an UnsupportedOperationException. All access is intended to be through its static methods.
- **Scope:** As a static utility, ViewUtil has no instance lifecycle. Its methods and constants are available for the entire application lifetime after the class is loaded by the Java Virtual Machine.
- **Destruction:** The class is unloaded by the JVM upon application termination. No manual resource management is required.

## Internal State & Concurrency
- **State:** ViewUtil is entirely stateless. It contains no mutable or immutable fields, either static or instance. Each method operates exclusively on the arguments provided to it.
- **Thread Safety:** This class is inherently thread-safe. Its stateless, pure-function design ensures that it can be safely called from multiple threads concurrently without any need for external synchronization or internal locking mechanisms.

## API Surface
All methods are private and thus not part of a public contract. The following table describes the internal implementation for architectural clarity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeOutCode(double x, double y) | private static int | O(1) | Computes a 4-bit "outcode" for a point relative to the normalized view volume. Each bit corresponds to a region: TOP, BOTTOM, RIGHT, or LEFT. |
| CohenSutherlandLineClipAndDraw(...) | private static void | O(1) | Implements the Cohen-Sutherland algorithm to clip a line segment. **Warning:** This method is a misnomer; it does not draw. |

## Integration Patterns

### Standard Usage
Direct usage of this class is not possible due to its private methods. It is designed to be an internal implementation detail for another class within the `com.hypixel.hytale.math.shape` package. A hypothetical invocation from a controlling class would look like the following.

```java
// This is a hypothetical example.
// This code will not compile from an external package.
// It assumes a caller within the same class file.

class ShapeRenderer {
    public void drawLine(Graphics2D g, double x0, double y0, double x1, double y1) {
        // The renderer would delegate the clipping logic to ViewUtil
        // before attempting to draw.
        ViewUtil.CohenSutherlandLineClipAndDraw(x0, y0, x1, y1, g, 0, 0);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance with `new ViewUtil()`. This will fail with an UnsupportedOperationException. The class is not designed to be instantiated.
- **Reflection-Based Access:** Bypassing the private access modifiers using reflection is a severe anti-pattern. It breaks encapsulation and couples client code to an internal, unstable implementation that is likely to change.

## Data Pipeline
ViewUtil acts as a pure computational function within a larger data flow, typically a rendering pipeline. It does not manage data persistence or transport.

> Flow:
> Raw Line Coordinates -> **ViewUtil.computeOutCode** -> Trivial Reject/Accept Check -> **ViewUtil.CohenSutherlandLineClipAndDraw** -> Clipped Line Coordinates -> Downstream Renderer

