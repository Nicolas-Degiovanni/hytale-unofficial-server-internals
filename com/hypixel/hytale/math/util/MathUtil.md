---
description: Architectural reference for MathUtil
---

# MathUtil

**Package:** com.hypixel.hytale.math.util
**Type:** Utility

## Definition
```java
// Signature
public class MathUtil {
```

## Architecture & Concepts
The MathUtil class is a foundational, engine-wide utility library for mathematical operations. It serves as a centralized repository of static methods for common calculations that are either not present in the standard Java Math library or are specifically optimized for game development contexts, such as physics, rendering, and gameplay logic.

This class is designed to be completely stateless. It provides pure functions that operate solely on their input arguments, ensuring predictable and consistent behavior across the entire codebase. Its existence prevents the proliferation of duplicated or inconsistent mathematical logic in higher-level systems.

Key functional areas include:
*   **Floating-Point Precision:** Epsilon-based comparisons and clipping for handling floating-point inaccuracies.
*   **Performance Optimizations:** "Fast" versions of rounding, flooring, and ceiling operations which may trade extreme-edge-case precision for speed.
*   **Game-Specific Logic:** Linear interpolation (lerp), angle wrapping, and vector rotations.
*   **Data Packing:** Bitwise operations to pack multiple smaller integer values into single integer or long primitives, commonly used for network serialization or compact data storage in chunks.
*   **Random Number Generation:** A thread-safe interface for generating random numbers, crucial for concurrent systems like procedural generation or particle effects.

## Lifecycle & Ownership
As a static utility class, MathUtil does not follow a traditional object lifecycle.

*   **Creation:** The class is never instantiated. Its private constructor explicitly prevents the creation of MathUtil objects, enforcing its role as a static-only utility. The class is loaded into the JVM by the ClassLoader upon its first use.
*   **Scope:** Application-global. Once loaded, its static methods and constants are available for the entire lifetime of the application.
*   **Destruction:** The class is unloaded from the JVM only when the application terminates. No manual cleanup is required or possible.

## Internal State & Concurrency
*   **State:** MathUtil is entirely stateless. It contains no mutable instance or static fields. All members are either `static final` constants (immutable) or `static` methods.
*   **Thread Safety:** This class is inherently thread-safe. All methods are pure functions that do not modify shared state. The random number generation methods specifically use `java.util.concurrent.ThreadLocalRandom` to avoid contention and ensure safe use in multi-threaded environments, such as parallel world generation or server-side logic processing. It can be safely called from any thread without synchronization.

## API Surface
The following is a representative subset of the most critical functions provided by MathUtil.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| clamp(v, min, max) | T | O(1) | Constrains a value of type T (int, float, double, long) to be within the specified min/max range. |
| lerp(a, b, t) | float/double | O(1) | Performs linear interpolation between two values. The factor t is clamped to the range [0.0, 1.0]. |
| fastFloor(f) | int | O(1) | A high-performance alternative to Math.floor. May have different behavior for edge-case values. |
| packInt(x, z) | int | O(1) | Packs two 16-bit integer values into a single 32-bit integer using bitwise operations. |
| unpackLeft(packed) | int | O(1) | Extracts the high-order 16 bits from an integer packed with packInt. |
| rotateVectorYAxis(vector, angle, clockwise) | Vector3i/d | O(1) | Rotates a 3D vector around the Y-axis. Returns a new vector instance. |
| randomInt(min, max) | int | O(1) | Generates a thread-safe random integer within the specified bounds. |
| closeToZero(v, epsilon) | boolean | O(1) | Checks if a floating-point value is within a given epsilon of zero, avoiding direct equality checks. |

## Integration Patterns

### Standard Usage
MathUtil is designed for direct static invocation. It should be used anywhere a common mathematical operation is required.

```java
// Example: Clamping player health
int maxHealth = 100;
int currentHealth = 120;
currentHealth = MathUtil.clamp(currentHealth, 0, maxHealth); // currentHealth is now 100

// Example: Interpolating an entity's position for smooth rendering
float renderX = MathUtil.lerp(previousX, currentX, partialTick);
```

### Anti-Patterns (Do NOT do this)
*   **Attempted Instantiation:** Never attempt to create an instance of MathUtil. The private constructor makes this impossible through normal means, and circumventing it via reflection serves no purpose and violates the class design.
*   **Logic Duplication:** Do not re-implement functions like `clamp` or `lerp` in other parts of the codebase. Always defer to this centralized utility to maintain consistency and correctness.
*   **Misuse of Fast Functions:** The `fastFloor`, `fastCeil`, and `fastRound` methods are intended for performance-critical code paths where the input domain is well-known. They may not provide the same guarantees as the standard `java.lang.Math` equivalents for all possible inputs. Avoid using them unless a performance bottleneck has been identified.

## Data Pipeline
MathUtil is not a component within a data pipeline; rather, it is a foundational toolkit used by components at every stage of various pipelines. It does not ingest, transform, or output data in a sequential flow. Instead, its functions are atomic operations applied to data as it moves through other systems.

> **Example Flow (Animation System):**
> Keyframe Data -> **MathUtil.lerp** -> Interpolated Bone Transform -> Renderer

> **Example Flow (Physics Engine):**
> Velocity Vector -> Collision Check -> **MathUtil.clamp** -> Final Position -> World State Update

