---
description: Architectural reference for TrigMathUtil
---

# TrigMathUtil

**Package:** com.hypixel.hytale.math.util
**Type:** Utility

## Definition
```java
// Signature
public class TrigMathUtil {
```

## Architecture & Concepts
TrigMathUtil is a high-performance, low-precision replacement for the standard Java Math library's trigonometric functions. Its primary role within the engine is to accelerate the immense volume of trigonometric calculations required by the physics and rendering systems, where execution speed is more critical than absolute mathematical precision.

This class achieves its performance by implementing a classic game development optimization: the use of pre-computed Look-Up Tables (LUTs). Instead of performing expensive transcendental function calls for sine, cosine, and arctangent, it approximates these values by retrieving them from static arrays initialized at startup.

The implementation is encapsulated within two private static inner classes:
*   **Riven:** Manages the sine and cosine LUTs. It maps a radian input to an index in a 4096-element array.
*   **Icecore:** Manages the arctangent (atan2) LUT. It uses a larger, 100,001-element table and quadrant-specific logic to approximate the result.

This design pattern trades a small amount of memory and a one-time startup cost for a significant reduction in CPU load during the main game loop.

### Lifecycle & Ownership
- **Creation:** As a static utility class, TrigMathUtil is never instantiated. Its internal state, the static LUTs, is initialized by the Java ClassLoader when the class is first referenced by any part of the engine. This initialization occurs once, typically very early in the application's bootstrap sequence.
- **Scope:** The class and its associated lookup tables persist in memory for the entire lifetime of the application.
- **Destruction:** The memory is reclaimed by the JVM only upon application shutdown. No manual cleanup is necessary or possible.

## Internal State & Concurrency
- **State:** The internal state consists of three static final float arrays: SIN and COS within the Riven class, and ATAN2 within the Icecore class. After the static initializer blocks complete during class loading, this state is **effectively immutable**. The tables are populated once and are never modified during runtime.
- **Thread Safety:** This class is unconditionally **thread-safe**. All public methods are pure, stateless functions that only perform read operations on the immutable, pre-computed tables. No synchronization or locking mechanisms are required, making it safe for high-throughput, concurrent access from any engine thread, including the main logic, rendering, and physics threads.

## API Surface
The public API provides low-precision, high-speed alternatives to standard math functions.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| sin(float radians) | float | O(1) | Returns the approximated sine of an angle. |
| cos(float radians) | float | O(1) | Returns the approximated cosine of an angle. |
| atan2(float y, float x) | float | O(1) | Returns the approximated angle theta from the conversion of rectangular coordinates (x, y) to polar coordinates (r, theta). |

**WARNING:** The results from these methods are approximations. Do not use them in contexts where high precision is required. The margin of error is a direct result of the LUT resolution.

## Integration Patterns

### Standard Usage
This class should be used via static calls in performance-critical code paths that involve frequent geometric or rotational calculations.

```java
// Example: Updating entity orientation in the game loop
float orientation = TrigMathUtil.atan2(velocity.y, velocity.x);
float facingX = TrigMathUtil.cos(orientation);
float facingZ = TrigMathUtil.sin(orientation);
```

### Anti-Patterns (Do NOT do this)
- **Precision-Critical Calculations:** Do not use this utility for any system where mathematical accuracy is paramount, such as serialization of financial data, scientific modeling, or complex procedural generation algorithms that are sensitive to floating-point drift. Use the standard java.lang.Math library for such cases.
- **Attempted Instantiation:** The class has a private constructor and cannot be instantiated. All usage must be through static method calls (e.g., TrigMathUtil.sin(...)).

## Data Pipeline
The data flow for a function call is a direct, non-blocking lookup. The input value is transformed into an array index, and the pre-computed result is returned.

> Flow (for sin):
> Input Radian (float) -> Index Calculation (`(int)(rad * radToIndex) & SIN_MASK`) -> **Riven.SIN[]** -> Approximated Result (float)

> Flow (for atan2):
> Input Y/X (float) -> Quadrant Logic & Index Calculation -> **Icecore.ATAN2[]** -> Approximated Result (float)

