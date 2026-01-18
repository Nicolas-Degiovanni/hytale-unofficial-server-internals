---
description: Architectural reference for ExponentialResponseCurve
---

# ExponentialResponseCurve

**Package:** com.hypixel.hytale.server.core.asset.type.responsecurve.config
**Type:** Configuration Model / Transient

## Definition
```java
// Signature
public class ExponentialResponseCurve extends ResponseCurve {
```

## Architecture & Concepts
The ExponentialResponseCurve is a concrete implementation of the ResponseCurve abstraction. Its purpose is to model a configurable mathematical curve based on the exponential function `y = m * (x - h)^e + v`. This allows game designers to define non-linear relationships between an input and an output in a declarative way.

This class is fundamentally a data-driven component. It is not intended for procedural construction within game logic. Instead, its parameters—slope, exponent, and shifts—are defined within external asset files, typically JSON. The static field CODEC is the critical integration point with the engine's serialization framework. This codec is responsible for parsing the asset data and hydrating an in-memory instance of this class.

Common use cases include:
*   **AI Behavior:** Mapping an enemy's distance to a target to its aggression level.
*   **Audio Attenuation:** Defining a custom falloff curve for sound volume over distance.
*   **Gameplay Mechanics:** Calculating weapon recoil recovery speed over time.
*   **Visual Effects:** Controlling the acceleration or opacity of a particle effect over its lifetime.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the engine's asset loading pipeline. The AssetManager reads a source file, and the Codec system uses the static CODEC definition within this class to deserialize the data and construct a new ExponentialResponseCurve object.

- **Scope:** The lifetime of an instance is bound to the lifetime of the parent asset that contains it. For example, if a curve defines a property of a specific monster type, the curve object will be loaded when the monster asset is loaded and will persist in memory as long as that monster definition is required.

- **Destruction:** Instances are managed by the Java Garbage Collector. They are eligible for cleanup once the owning asset is unloaded and all references to the curve object are released. There are no explicit destruction or teardown methods.

## Internal State & Concurrency
- **State:** The object's state (slope, exponent, horizontalShift, verticalShift) is mutable during its construction by the Codec builder. After deserialization is complete, the instance should be treated as **effectively immutable**. Its purpose is to hold configuration data, not to track dynamic state.

- **Thread Safety:** The class is **conditionally thread-safe**. The primary method, computeY, is a pure function that only reads from instance fields. It does not modify internal state and has no side effects. Therefore, a fully constructed instance can be safely shared and read by multiple threads simultaneously without external synchronization.

    **WARNING:** Modifying the state of a shared instance from any thread after its initial creation is not a supported operation and will lead to race conditions and non-deterministic behavior.

## API Surface
The public API is minimal, focusing entirely on retrieving the computed value from the curve.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeY(double x) | double | O(1) | Calculates the output value Y for a given input X. Throws IllegalArgumentException if X is outside the normalized range of 0.0 to 1.0. |
| getSlope() | double | O(1) | Returns the configured slope of the curve. |
| getExponent() | double | O(1) | Returns the configured exponent of the curve. |
| getHorizontalShift() | double | O(1) | Returns the configured horizontal shift of the curve. |
| getVerticalShift() | double | O(1) | Returns the configured vertical shift of the curve. |

## Integration Patterns

### Standard Usage
The intended pattern is to retrieve a pre-configured instance from a parent asset and use it for calculations. The curve itself is defined entirely within a data file.

**Example Asset Definition (conceptual JSON):**
```json
{
  "id": "my_weapon_recoil",
  "type": "ExponentialResponseCurve",
  "slope": 0.8,
  "exponent": 2.5,
  "horizontalShift": 0.1,
  "verticalShift": 0.0
}
```

**Example Consumption Code:**
```java
// Assume 'weaponData' is an asset object that holds a ResponseCurve
ResponseCurve recoilCurve = weaponData.getRecoilCurve();

// Calculate the recoil multiplier at the halfway point of the animation
double time = 0.5;
double recoilMultiplier = recoilCurve.computeY(time);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Manually constructing an ExponentialResponseCurve with `new` is a critical anti-pattern. This embeds gameplay-critical "magic numbers" directly into code, making them difficult for designers to tune and bypassing the entire data-driven asset system.

    ```java
    // DO NOT DO THIS
    ResponseCurve curve = new ExponentialResponseCurve(0.8, 2.5, 0.1, 0.0); // BAD: Hardcoded values
    ```

- **Post-Creation State Mutation:** Modifying the fields of a curve after it has been loaded from an asset is a violation of its design contract. Assets are considered shared, immutable data sources. Such modifications can cause unpredictable side effects in other systems that hold a reference to the same instance.

## Data Pipeline
The flow of data from configuration to utilization is linear and unidirectional.

> Flow:
> JSON Asset File -> AssetManager -> Codec Deserializer -> **ExponentialResponseCurve Instance** -> Game System (e.g., AI, Physics) -> Calculated Value

