---
description: Architectural reference for SineWaveResponseCurve
---

# SineWaveResponseCurve

**Package:** com.hypixel.hytale.server.core.asset.type.responsecurve.config
**Type:** Configuration Model

## Definition
```java
// Signature
public class SineWaveResponseCurve extends ResponseCurve {
```

## Architecture & Concepts
The **SineWaveResponseCurve** is a specific, data-driven implementation of the abstract **ResponseCurve** concept. Its primary architectural role is to provide a stateless function that transforms a normalized linear input (a value between 0.0 and 1.0) into a non-linear, oscillating output according to a sine wave formula.

This class is not a service or a manager; it is a plain data object whose behavior is entirely defined by its configuration. It is designed to be deserialized from asset files (e.g., JSON) via the Hytale **Codec** system. This pattern allows game designers and content creators to define complex mathematical behaviors—such as procedural animation curves, entity behavior falloffs, or world generation patterns—declaratively in data files, without requiring direct code changes.

It is a fundamental building block in systems that require smooth, periodic, or natural-feeling value modulation.

## Lifecycle & Ownership
-   **Creation:** Instances are almost exclusively created by the Hytale **Codec** system during the asset loading phase. The public static final **CODEC** field acts as the factory and deserializer, parsing configuration data and populating a new **SineWaveResponseCurve** object. Manual instantiation is strongly discouraged.
-   **Scope:** The lifetime of a **SineWaveResponseCurve** instance is tied to the asset that defines it. It is typically loaded into memory by an asset management system and cached. It persists as long as the parent asset is considered active, which could be for the duration of a game session or world load.
-   **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection once it is no longer referenced by any asset cache or game system.

## Internal State & Concurrency
-   **State:** The internal state consists of four double-precision floating-point numbers: **amplitude**, **frequency**, **horizontalShift**, and **verticalShift**. This state is mutable *only* during the deserialization process managed by the **CODEC**. After instantiation, the object is effectively immutable, as no public methods exist to alter its internal state.
-   **Thread Safety:** The object is thread-safe for all read operations after its initial creation. The primary method, **computeY**, is a pure function that depends only on its input and the immutable internal state. It can be safely called from multiple threads simultaneously without locks or synchronization, making it highly suitable for performance-critical, parallelized systems like procedural generation or physics updates.

## API Surface
The public contract is minimal, focusing on a single computation method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeY(double x) | double | O(1) | Computes the Y value for a given X based on the sine wave parameters. Throws IllegalArgumentException if X is not in the range [0.0, 1.0]. |
| getAmplitude() | double | O(1) | Returns the configured amplitude of the wave. |
| getFrequency() | double | O(1) | Returns the configured frequency of the wave. |
| getHorizontalShift() | double | O(1) | Returns the configured horizontal phase shift. |
| getVerticalShift() | double | O(1) | Returns the configured vertical offset. |

## Integration Patterns

### Standard Usage
A **SineWaveResponseCurve** is not used directly but is retrieved as a generic **ResponseCurve** from a parent asset or configuration object. The calling system is concerned with the abstract behavior, not the specific implementation.

```java
// Assume 'someAsset' is an object loaded from a config file
// that contains a ResponseCurve definition.
ResponseCurve curve = someAsset.getAnimationCurve();

// Use the curve to modulate a value over time (e.g., a normalized tick counter)
double normalizedTime = 0.75;
double modulatedValue = curve.computeY(normalizedTime);

// Apply the result to a game object property
entity.setOpacity(modulatedValue);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new SineWaveResponseCurve()`. The constructor is protected to prevent this. Bypassing the **CODEC** system results in an object with default, unconfigured parameters, which is almost never the desired behavior.
-   **State Assumption:** Do not assume the default values. The behavior of any given instance is entirely dependent on the data from which it was loaded. Always treat it as a black box configured by designers.

## Data Pipeline
The class is a destination in the asset loading pipeline and a source of computed values for the game logic pipeline.

> **Asset Loading Flow:**
> Asset File on Disk (e.g., JSON) -> Hytale Asset Loader -> **Codec Deserializer** -> **SineWaveResponseCurve Instance** (in memory)
>
> **Game Logic Flow:**
> Game System (e.g., Animator) -> **computeY(x)** -> Modulated Value -> Game State Update (e.g., Bone Position)

