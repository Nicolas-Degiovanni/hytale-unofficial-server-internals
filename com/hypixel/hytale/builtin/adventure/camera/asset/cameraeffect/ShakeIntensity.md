---
description: Architectural reference for ShakeIntensity
---

# ShakeIntensity

**Package:** com.hypixel.hytale.builtin.adventure.camera.asset.cameraeffect
**Type:** Data Model / Configuration Object

## Definition
```java
// Signature
public class ShakeIntensity {
```

## Architecture & Concepts

The ShakeIntensity class is a data-driven configuration object that defines the magnitude and behavior of a camera shake effect. It is not a service or a manager; rather, it serves as a data model that is deserialized from asset files, typically JSON. This design decouples game feel—specifically camera feedback—from game logic, allowing designers to tune effects without modifying engine code.

Its primary role is to translate a game event's context, such as damage taken or explosion proximity, into a concrete intensity value used by the Camera System. This is achieved through three core properties:

1.  **Base Value:** A static float value used as a default intensity when no specific context is provided.
2.  **Accumulation Mode:** An enumeration that dictates how this shake effect's intensity combines with other active shake effects of the same type. This is a critical component for preventing visual chaos when multiple events occur simultaneously, enabling strategies like taking the maximum intensity, summing intensities, or having a new effect replace an old one.
3.  **Modifier:** A powerful, optional mechanism for dynamic intensity calculation. The nested Modifier class implements a piecewise linear function, mapping an input range (e.g., 0-100 damage) to an output range (e.g., 0.0-1.0 shake intensity). This provides a designer-controlled response curve, allowing for non-linear relationships between game events and their perceived impact.

This class is fundamentally tied to the engine's serialization framework via its static **CODEC** field. It is intended to be defined entirely within data files and loaded at runtime by the asset management system.

### Lifecycle & Ownership

-   **Creation:** ShakeIntensity instances are created exclusively by the engine's **BuilderCodec** during the asset loading phase. Game systems do not instantiate this class directly. The codec reads a definition from an asset file (e.g., a camera effect JSON) and populates a new ShakeIntensity object.
-   **Scope:** An instance of ShakeIntensity has a scope tied to the asset it is part of. Once loaded, it exists as a read-only template in an asset registry for the duration of the game session or until its asset package is unloaded.
-   **Destruction:** The object is marked for garbage collection when the associated assets are unloaded, for instance, when the client returns to the main menu or shuts down.

## Internal State & Concurrency

-   **State:** The internal state (value, accumulationMode, modifier) is mutable only during its construction by the BuilderCodec. After deserialization is complete, the object should be treated as **effectively immutable**. Game logic must not attempt to modify its state at runtime.
-   **Thread Safety:** The object is **thread-safe for all read operations**. Because its state is fixed after loading, multiple systems (e.g., physics, rendering, game logic) can safely access its properties concurrently without locks. The Modifier.apply method is a pure function, guaranteeing that it produces the same output for a given input without side effects.

## API Surface

The public API is minimal, designed for read-only access to the configured properties.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | float | O(1) | Returns the static, base intensity value. |
| getAccumulationMode() | AccumulationMode | O(1) | Returns the configured strategy for combining this effect with others. |
| getModifier() | ShakeIntensity.Modifier | O(1) | Returns the optional modifier for dynamic intensity calculation. May be null. |
| Modifier.apply(float) | float | O(N) | Calculates a new intensity based on a contextual input. N is the number of points in the modifier's lookup table. |

## Integration Patterns

### Standard Usage

The ShakeIntensity object is always used as part of a larger, pre-loaded asset, such as a CameraEffect. A game system retrieves the effect and uses its ShakeIntensity configuration to calculate the final magnitude before passing it to the Camera System.

```java
// Assume 'damageEffect' is a CameraEffect asset loaded from a registry
// Assume 'damageAmount' is the contextual value from a game event

// Retrieve the intensity configuration from the parent effect
ShakeIntensity intensityConfig = damageEffect.getIntensity();
float finalIntensity = intensityConfig.getValue(); // Start with the base value

// If a modifier is present, use it to calculate a dynamic intensity
ShakeIntensity.Modifier modifier = intensityConfig.getModifier();
if (modifier != null) {
    finalIntensity = modifier.apply(damageAmount);
}

// The camera system now uses finalIntensity and the accumulation mode
cameraSystem.applyShake(damageEffect.getId(), finalIntensity, intensityConfig.getAccumulationMode());
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ShakeIntensity()`. This bypasses the entire data-driven asset pipeline and results in an unconfigured, useless object. All definitions must reside in asset files.
-   **Runtime Modification:** Do not attempt to modify the state of a ShakeIntensity object after it has been loaded. This violates the "asset as template" contract and can lead to unpredictable, global side effects.
-   **Ignoring Null Modifier:** The `getModifier()` method can return null. Code must be robust and fall back to using `getValue()` in this scenario, otherwise a NullPointerException will occur.

## Data Pipeline

The flow of data for ShakeIntensity begins with a designer's definition and ends with a pixel-level effect on the player's screen.

> Flow:
> Designer Asset (JSON) -> Engine Asset Loader -> **ShakeIntensity.CODEC** -> In-Memory ShakeIntensity Object -> Game Event (e.g., Player Damaged) -> Effect System reads ShakeIntensity -> Calculated float value -> Camera System -> Screen Shake
---
# ShakeIntensity.Modifier

**Package:** com.hypixel.hytale.builtin.adventure.camera.asset.cameraeffect
**Type:** Data Model / Utility

## Definition
```java
// Signature
public static class Modifier implements FloatUnaryOperator {
```

## Architecture & Concepts

The Modifier is a nested static class within ShakeIntensity that provides a powerful, data-driven mechanism for mapping a contextual game value to a camera shake intensity. It functions as a configurable lookup table (LUT) or a piecewise linear function, allowing designers to create non-linear response curves for game feedback.

Architecturally, it serves as a pure function (`FloatUnaryOperator`) that takes a single float input (e.g., damage amount, explosion force) and produces a float output (the final shake intensity). This is defined by two parallel arrays, **input** and **output**, which are deserialized from asset files.

For example, a designer can specify that damage from 0 to 10 maps to a shake intensity of 0.0 to 0.2, while damage from 10 to 100 maps to an intensity of 0.2 to 1.0. The `apply` method performs a linear scan to find the correct segment in the input array and then uses linear interpolation (**MathUtil.mapToRange**) to calculate the precise output value. This system is highly efficient and provides fine-grained control over the "feel" of an effect.

### Lifecycle & Ownership

-   **Creation:** A Modifier is created by the **BuilderCodec** as part of its parent ShakeIntensity object's deserialization. It is never instantiated directly by game code.
-   **Scope:** Its lifecycle is identical to and strictly owned by its parent ShakeIntensity instance. It exists as long as the parent asset is loaded in memory.
-   **Destruction:** It is garbage collected when its parent ShakeIntensity is collected.

## Internal State & Concurrency

-   **State:** The `input` and `output` arrays are populated once during deserialization and are treated as **immutable** thereafter.
-   **Thread Safety:** The Modifier class is **inherently thread-safe**. The `apply` method is a pure, stateless function that only reads from its internal arrays. It can be called from any thread without synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(float intensityContext) | float | O(N) | Maps the input `intensityContext` to an output intensity value by interpolating between points defined in the input and output arrays. N is the number of points in the arrays. |

## Integration Patterns

### Standard Usage

The Modifier is never used in isolation. It is always accessed through its parent ShakeIntensity object. See the standard usage pattern for ShakeIntensity for a complete example.

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** As with its parent, do not use `new ShakeIntensity.Modifier()`. This object must be configured via asset files.
-   **Invalid Array Configuration:** In the asset definition, the `input` array values must be monotonically increasing. Failure to do so will result in incorrect or unpredictable behavior from the `apply` method. The engine's validators attempt to catch this, but it remains a critical data integrity constraint.

