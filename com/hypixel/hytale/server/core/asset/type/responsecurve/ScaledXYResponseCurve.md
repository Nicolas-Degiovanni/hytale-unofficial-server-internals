---
description: Architectural reference for ScaledXYResponseCurve
---

# ScaledXYResponseCurve

**Package:** com.hypixel.hytale.server.core.asset.type.responsecurve
**Type:** Transient Data Model

## Definition
```java
// Signature
public class ScaledXYResponseCurve extends ScaledXResponseCurve {
```

## Architecture & Concepts
The ScaledXYResponseCurve is a data-driven mathematical primitive used throughout the engine to map an input value to a corresponding output value, with scaling applied to both axes. It represents a function *f(x) = y* where the input domain (the X-axis) and the output range (the Y-axis) are remapped from a normalized space to a user-defined range.

This class extends ScaledXResponseCurve, inheriting the logic for scaling the input X-axis. Its unique responsibility is to add the final transformation step of scaling the output Y-axis.

Its primary role is to decouple game logic from hard-coded values, allowing designers to define complex behaviors declaratively in asset files. For example, a ScaledXYResponseCurve can define how a monster's damage output scales with its health, how environmental fog density changes with altitude, or how the speed of a particle effect changes over its lifetime.

The static CODEC field is the most critical architectural feature. It signals that this class is a component of the engine's serialization framework. Instances are not typically created imperatively in code but are instead deserialized from asset files by the Hytale Asset Loader.

## Lifecycle & Ownership
-   **Creation:** Instances are almost exclusively created by the engine's asset loading system during server or client startup when parsing asset files. The system uses the public static CODEC field to deserialize the data into a fully-formed ScaledXYResponseCurve object. Manual instantiation is possible but strongly discouraged.
-   **Scope:** The lifetime of a ScaledXYResponseCurve instance is bound to the lifetime of the containing asset. For example, if a curve defines a property of a specific block, the curve object will remain in memory as long as that block's definition is loaded. It is not a global or session-scoped object.
-   **Destruction:** The object is managed by the Java Garbage Collector. It is eligible for collection once the parent asset that owns it is unloaded and no other systems hold a reference to it. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
-   **State:** The internal state, consisting of xRange (inherited) and yRange, is mutable upon construction. However, after deserialization and injection into a game system, instances should be treated as **effectively immutable**. The design contract assumes that its range parameters will not change during its lifetime.
-   **Thread Safety:** This class is **conditionally thread-safe**. The core computeY method is a pure function with no side effects and does not modify internal state. Therefore, multiple threads can safely call computeY on a shared instance concurrently without external locking, provided the underlying arrays are not modified after construction.

## API Surface
The public API is minimal, focusing exclusively on the core transformation logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeY(double x) | double | O(1) | Computes the scaled output Y for a given input X. The complexity assumes the parent's normalized computation is also constant time. Clamps the result to the defined Y-Range. |
| getYRange() | double[] | O(1) | Returns a direct reference to the internal yRange array. **Warning:** Modifying this returned array is an anti-pattern and will break thread-safety guarantees. |

## Integration Patterns

### Standard Usage
The intended use is declarative. A designer defines the curve in an asset file, and a game system retrieves the fully-formed object from the asset to perform calculations.

```java
// In a game system that has access to a loaded asset
// (e.g., a monster's definition)

// 1. Retrieve the pre-configured curve from the asset
ResponseCurve curve = monsterDefinition.getDamageScaleCurve();

// 2. Use the curve to compute a value
double currentHealthRatio = 0.5; // Example input
double damageMultiplier = curve.computeY(currentHealthRatio);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Avoid creating instances with `new ScaledXYResponseCurve()`. This practice hard-codes configuration that should be managed in data files, creating tight coupling between logic and data and making balance changes difficult.
-   **State Mutation:** Do not modify the array returned by getYRange. This violates the "effectively immutable" contract and can lead to severe and difficult-to-debug race conditions if the instance is used by multiple systems or threads.

## Data Pipeline
The ScaledXYResponseCurve acts as a pure transformation step within a larger data flow, typically originating from asset files and influencing real-time game mechanics.

> Flow:
> Game Asset File (e.g., monster.json) -> Engine Asset Loader (using CODEC) -> **ScaledXYResponseCurve Instance** -> Game System (e.g., Combat Logic) -> `computeY(input)` -> Scaled Game Parameter (e.g., Damage Value)

