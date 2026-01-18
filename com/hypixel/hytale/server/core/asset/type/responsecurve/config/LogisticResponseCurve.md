---
description: Architectural reference for LogisticResponseCurve
---

# LogisticResponseCurve

**Package:** com.hypixel.hytale.server.core.asset.type.responsecurve.config
**Type:** Transient Data Model

## Definition
```java
// Signature
public class LogisticResponseCurve extends ResponseCurve {
```

## Architecture & Concepts
The LogisticResponseCurve is a specific implementation of the ResponseCurve strategy. It models a mathematical logistic function, commonly known as an S-curve. Its primary role within the engine is to provide game designers with a powerful, data-driven tool for non-linear value mapping.

This is frequently used for game balancing mechanics, such as:
*   Mapping player level to damage output, where gains diminish at higher levels.
*   Controlling the rate of resource generation over time.
*   Defining armor effectiveness against incoming damage, providing an initial high resistance that falls off.

The most critical architectural feature is the static **CODEC** field. This is not merely a helper; it is the contract that integrates this class with Hytale's asset loading and configuration system. The BuilderCodec declaratively defines how a textual representation (e.g., JSON) of a logistic curve is deserialized into a valid Java object. This allows designers to define and tweak complex mathematical behaviors in data files without requiring code changes.

## Lifecycle & Ownership
- **Creation:** LogisticResponseCurve instances are almost exclusively instantiated by the Hytale **Codec** system during the asset loading phase. A game designer defines the curve's parameters in a configuration file, and the engine's AssetManager uses the public static CODEC to construct the object in memory. Programmatic creation via its constructor is possible but is reserved for unit testing or dynamic, in-memory generation.

- **Scope:** The object's lifetime is bound to the parent asset or component that holds a reference to it. For example, if a curve defines a weapon's damage falloff, the LogisticResponseCurve instance will exist as long as the weapon's configuration data is loaded. It is a value object, not a managed service.

- **Destruction:** Instances are managed by the Java Garbage Collector. There are no manual cleanup or disposal methods. They are eligible for collection once all references from game systems or asset caches are released.

## Internal State & Concurrency
- **State:** The object's state is defined by its four protected double fields: rateOfChange, ceiling, horizontalShift, and verticalShift. The class is **mutable**, as these fields can be modified after construction. However, the design intent is for instances to be treated as **effectively immutable** after being loaded from configuration.

- **Thread Safety:** This class is **not thread-safe**. Its fields are accessed directly without any synchronization. Concurrent calls to computeY while another thread modifies a parameter (e.g., rateOfChange) will lead to undefined behavior and calculation errors.

    **Warning:** All access and modification must be externally synchronized if the object is to be shared and mutated across threads. The standard pattern is to load it once on the main thread and then treat it as a read-only object.

## API Surface
The public contract is focused on configuration via the CODEC and the core computation method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | O(1) | **Critical.** Public static field used by the engine to serialize and deserialize the object from asset files. |
| computeY(double x) | double | O(1) | Computes the Y-value for a given X-value on the curve. Throws IllegalArgumentException if X is not between 0.0 and 1.0. |

## Integration Patterns

### Standard Usage
The intended use is to retrieve a pre-configured instance from a larger asset or configuration object and use it to transform a normalized input value.

```java
// Assume 'weaponConfig' is an object loaded from game assets
ResponseCurve damageFalloffCurve = weaponConfig.getDamageFalloffCurve();

// Normalize distance to a 0.0-1.0 range
double normalizedDistance = currentDistance / maxWeaponRange;

// Compute the damage multiplier using the curve
double damageMultiplier = damageFalloffCurve.computeY(normalizedDistance);
double finalDamage = baseDamage * damageMultiplier;
```

### Anti-Patterns (Do NOT do this)
- **Runtime Modification:** Do not modify the parameters of a shared LogisticResponseCurve instance at runtime. This breaks the "effectively immutable" contract and is not thread-safe. If a different curve is needed, load or create a new instance.
- **Direct Instantiation:** Avoid using `new LogisticResponseCurve()` in game logic. This hard-codes balancing values. Curves should be defined in data files and loaded via the AssetManager to empower game designers.
- **Unsafe Concurrent Access:** Do not call `computeY` from one thread while another thread might be modifying the object's parameters. This will lead to data races.

## Data Pipeline
The LogisticResponseCurve acts as a configuration endpoint in the asset loading pipeline. It does not process streaming data; rather, it is the *result* of a data loading process.

> Flow:
> Game Asset File (JSON) -> Engine AssetManager -> Hytale Codec System -> **LogisticResponseCurve Instance** -> Game System (e.g., CombatService) for computation

