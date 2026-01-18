---
description: Architectural reference for ScaledResponseCurve
---

# ScaledResponseCurve

**Package:** com.hypixel.hytale.server.core.asset.type.responsecurve
**Type:** Abstract Asset

## Definition
```java
// Signature
public abstract class ScaledResponseCurve implements JsonAssetWithMap<String, DefaultAssetMap<String, ScaledResponseCurve>> {
```

## Architecture & Concepts
The ScaledResponseCurve is a foundational abstract class that defines a contract for mapping a single floating-point input value to a corresponding output value. It serves as the polymorphic base for a data-driven system that allows game designers to define complex mathematical relationships in JSON asset files instead of hardcoding them in source code.

Its primary architectural role is to integrate with the Hytale asset loading pipeline. The static **CODEC** field is the central component, acting as a polymorphic factory. This codec is responsible for deserializing JSON data and instantiating the correct concrete subclass (e.g., ScaledXResponseCurve, ScaledSwitchResponseCurve) based on a type identifier specified in the asset file.

This system is critical for features requiring tunable, non-linear progressions, such as damage falloff over distance, experience requirements per level, or AI response intensity based on a stimulus.

### Lifecycle & Ownership
- **Creation:** Instances are not created directly using the **new** keyword. They are instantiated exclusively by the asset loading system during server or client bootstrap. The static **CODEC** field is invoked by the AssetStore to deserialize the corresponding JSON file into a concrete ScaledResponseCurve object.
- **Scope:** The lifecycle of a ScaledResponseCurve instance is tied to the asset cache. Once loaded, it persists in memory for the entire duration of the server session or until the asset cache is explicitly cleared.
- **Destruction:** Instances are marked for garbage collection when the AssetManager is cleared or when all systems release their references to the object, typically during a world unload or server shutdown.

## Internal State & Concurrency
- **State:** Instances are effectively immutable after creation. Their internal parameters, which define the shape of the curve, are loaded from JSON files and are not designed to be modified at runtime. The protected **id** and **data** fields are set during deserialization and remain constant.
- **Thread Safety:** This class is inherently thread-safe for read operations. Because its state is immutable post-initialization, the core **computeY** method can be safely called from multiple threads concurrently without external locking.

    **Warning:** This thread-safety guarantee relies on all subclasses also being immutable. Introducing mutable state in a concrete implementation would violate the architectural contract and is strongly discouraged.

## API Surface
The public contract is minimal, focusing entirely on the core function of the curve.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeY(double var1) | double | Varies | **Core Method.** Computes the output Y for a given input X. Complexity depends on the subclass implementation (e.g., O(1) for linear, O(log n) for a stepped curve). |
| getId() | String | O(1) | Returns the unique asset identifier for this curve. |

## Integration Patterns

### Standard Usage
A game system retrieves a pre-loaded curve from the asset registry and uses it to compute a game value. The caller should be entirely agnostic to the concrete type of the curve.

```java
// Example: Calculating damage falloff in a combat system
// Assume 'assetManager' is an available service
ScaledResponseCurve falloffCurve = assetManager.get("hytale:weapon_curves/pistol_falloff");
double baseDamage = 50.0;
double distance = 25.5;

// The curve computes the damage multiplier based on distance
double multiplier = falloffCurve.computeY(distance);
double finalDamage = baseDamage * multiplier;
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ScaledXResponseCurve()`. This bypasses the asset pipeline, resulting in an unmanaged object that will not be registered with game systems and will lack critical metadata.
- **Conditional Type Casting:** Avoid checking the specific type of the curve. The system is designed for polymorphism. Code should rely only on the abstract **computeY** contract.

    ```java
    // BAD: Violates polymorphism
    if (curve instanceof ScaledSwitchResponseCurve) {
        // ... custom logic
    }
    ```

## Data Pipeline
The ScaledResponseCurve is not a data processor itself; rather, it is the *result* of a data pipeline. The flow below describes its creation from source assets.

> Flow:
> JSON Asset File (`.json`) -> Hytale AssetStore -> Jackson Deserializer -> **ScaledResponseCurve.CODEC** -> Instantiated Concrete Subclass -> AssetManager Cache

