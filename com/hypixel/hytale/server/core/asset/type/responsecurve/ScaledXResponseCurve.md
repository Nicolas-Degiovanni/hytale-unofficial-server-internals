---
description: Architectural reference for ScaledXResponseCurve
---

# ScaledXResponseCurve

**Package:** com.hypixel.hytale.server.core.asset.type.responsecurve
**Type:** Transient Data Asset

## Definition
```java
// Signature
public class ScaledXResponseCurve extends ScaledResponseCurve {
```

## Architecture & Concepts
The ScaledXResponseCurve is a data-driven mathematical adapter. Its primary function is to take a standard, normalized ResponseCurve asset (which operates on an input domain of 0.0 to 1.0) and remap its horizontal axis to an arbitrary, user-defined range.

This class acts as a configuration-driven wrapper, allowing designers to reuse a single base curve for multiple scenarios with different input scales without requiring code changes. For example, a single "ease-in" curve can be scaled to model player acceleration from 0 to 100 m/s, or a weapon's damage falloff from 5 to 50 meters.

Instantiation and configuration are managed exclusively through the Hytale asset pipeline via the static CODEC field. This codec is responsible for deserializing the asset definition from a configuration file, validating its structure, and performing post-load processing to resolve asset references.

**WARNING:** This class is not a general-purpose service. It is a passive data structure whose logic is invoked by other game systems.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale asset loading system during server or client bootstrap. The static CODEC field is invoked to deserialize a corresponding asset file (e.g., a JSON file). The `afterDecode` hook within the codec is a critical final step that resolves the string-based `responseCurve` name into a direct `ResponseCurve.Reference`, preventing runtime lookups.
- **Scope:** An instance of ScaledXResponseCurve lives for the duration of the asset's lifecycle. It is loaded into a central asset registry and persists in memory until the registry is cleared or reloaded, such as during a server shutdown or a hot-reload event.
- **Destruction:** The object is managed by the Java Garbage Collector. It is eligible for collection once all references to it from the asset registry and any consuming systems are released. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** The internal state is effectively **immutable** after deserialization and the `afterDecode` hook completes. The fields `responseCurve`, `responseCurveReference`, and `xRange` are populated once during asset loading and are not intended to be modified at runtime.
- **Thread Safety:** This class is **thread-safe for all read operations**. Due to its immutable nature post-creation, multiple threads can safely and concurrently call the `computeY` method without locks or synchronization primitives. This makes it suitable for use in high-performance, multi-threaded systems like physics or AI behavior updates.

## API Surface
The public API is minimal, focusing on a single computational method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeY(double x) | double | O(1) | Computes the corresponding Y value for a given X value within the scaled range. The input X is clamped to the defined XRange. The complexity assumes the underlying ResponseCurve's computation is also O(1). |

## Integration Patterns
This component is designed to be defined as data and consumed by code. Direct manipulation or instantiation is a design violation.

### Standard Usage
A designer first defines the asset in a configuration file.

```json
// in some_asset_pack/assets/my_curves/weapon_falloff.json
{
  "__type": "ScaledXResponseCurve",
  "ResponseCurve": "hytale:curves/linear",
  "XRange": [ 10.0, 50.0 ]
}
```

A game system then retrieves the fully resolved asset from the asset manager and uses it for calculations.

```java
// In a game logic system
ResponseCurve falloffCurve = AssetManager.get(ResponseCurve.class, "my_curves:weapon_falloff");

// Calculate damage at 25 meters
double distance = 25.0;
double damageMultiplier = falloffCurve.computeY(distance);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ScaledXResponseCurve()`. This bypasses the asset pipeline and the critical `afterDecode` logic. The internal `responseCurveReference` will remain null, leading to a `NullPointerException` when `computeY` is called.
- **State Mutation:** Do not attempt to modify the `xRange` array after retrieving the asset. While the array itself is mutable, the system design assumes all assets are immutable. Modifying it can lead to unpredictable side effects in other systems that share a reference to the same asset.

## Data Pipeline
The flow of data for this component spans from design-time configuration to runtime execution.

> **Design & Load Time Flow:**
> 1. Designer creates a JSON asset file.
> 2. Hytale Asset Pipeline reads the file.
> 3. The **ScaledXResponseCurve.CODEC** deserializes the JSON into a raw object.
> 4. The `afterDecode` hook resolves the string "hytale:curves/linear" into a direct `ResponseCurve.Reference`.
> 5. The fully initialized object is stored in the Asset Registry.
>
> **Runtime Flow:**
> 1. Game System requests the asset by name from the Asset Registry.
> 2. The system calls `computeY` with an input value (e.g., 25.0).
> 3. **ScaledXResponseCurve** clamps the input and normalizes it from the `XRange` (e.g., [10, 50]) to the standard range [0, 1].
> 4. The normalized value is passed to the underlying base `ResponseCurve`'s `computeY` method.
> 5. The final Y value is returned to the Game System.

