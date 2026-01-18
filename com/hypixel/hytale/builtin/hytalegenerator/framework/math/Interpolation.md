---
description: Architectural reference for Interpolation
---

# Interpolation

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.math
**Type:** Utility

## Definition
```java
// Signature
public class Interpolation {
```

## Architecture & Concepts
The Interpolation class is a stateless, static utility providing foundational mathematical functions. It serves as a core component within the engine's math framework, offering pure functions for calculating intermediate values between two points.

This class is not part of a larger system but is a low-level dependency consumed by numerous higher-level systems. It is frequently leveraged by the animation system for keyframe blending, the physics engine for smoothing object motion, and procedural generation algorithms for creating continuous noise fields or terrain features. Its primary architectural role is to centralize and standardize common interpolation logic, preventing code duplication and ensuring consistent, predictable behavior across the engine.

## Lifecycle & Ownership
As a static utility class, Interpolation does not follow a traditional object lifecycle. It is never instantiated.

- **Creation:** The class is loaded into the JVM by the ClassLoader upon its first static reference. No instance is ever created.
- **Scope:** Application-wide. The static methods are available globally for the entire duration of the application's runtime.
- **Destruction:** The class is unloaded from the JVM when the application terminates. There is no manual cleanup or destruction process.

## Internal State & Concurrency
- **State:** The Interpolation class is entirely stateless. It contains no member variables, caches, or any other form of internal state. Each method call operates exclusively on the arguments provided.
- **Thread Safety:** This class is inherently thread-safe. All methods are pure functions with no side effects. It can be safely called from any number of concurrent threads without requiring locks or any other synchronization mechanisms.

## API Surface
The public contract consists of static, pure functions for mathematical calculations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| linear(valueA, valueB, weight) | double | O(1) | Calculates the linear interpolation between two values. Throws IllegalArgumentException if weight is not within the inclusive range of 0.0 to 1.0. |

## Integration Patterns

### Standard Usage
The class is designed for direct static invocation. It should be used whenever a smooth transition between two numerical values is required.

```java
// Example: Fading an object's opacity over time
double startOpacity = 1.0;
double endOpacity = 0.0;
double progress = 0.75; // 75% complete

double currentOpacity = Interpolation.linear(startOpacity, endOpacity, progress);
// Result: 0.25
```

### Anti-Patterns (Do NOT do this)
- **Attempted Instantiation:** The class is not designed to be instantiated. Do not use `new Interpolation()`. All methods are static.
- **Ignoring Input Constraints:** The `weight` parameter for the linear method has a strict contract of [0.0, 1.0]. Passing values outside this range will result in an exception. Callers are responsible for clamping or validating their inputs before invocation if necessary.
- **Logic Duplication:** Do not re-implement linear interpolation logic in other parts of the codebase. Use this centralized utility to maintain consistency and correctness.

## Data Pipeline
The Interpolation class acts as a pure transformation step within a larger data processing pipeline. It takes numerical inputs and produces a single numerical output without side effects.

> **Example Flow (Animation System):**
> Keyframe A Value -> Keyframe B Value -> Time Delta -> **Interpolation.linear** -> Final Blended Value -> Bone Transform Update

