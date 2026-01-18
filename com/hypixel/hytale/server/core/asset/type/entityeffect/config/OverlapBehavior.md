---
description: Architectural reference for OverlapBehavior
---

# OverlapBehavior

**Package:** com.hypixel.hytale.server.core.asset.type.entityeffect.config
**Type:** Enum / Utility

## Definition
```java
// Signature
public enum OverlapBehavior {
```

## Architecture & Concepts
The OverlapBehavior enum defines a set of policies that govern how the game engine resolves conflicts when an entity is subjected to a new status effect while already under the influence of an existing effect of the same type. This mechanism is a core component of the server-side Entity Effect System, providing a data-driven way to configure game mechanics without requiring code changes.

The presence of a static EnumCodec field is a critical architectural indicator. It signifies that this enum is not merely an internal engine constant but is designed to be serialized and deserialized from game asset files (e.g., JSON definitions for spells, potions, or environmental hazards). This allows game designers and modders to specify the desired behavior directly within an effect's configuration.

The three defined policies are:
*   **EXTEND:** The new effect's duration is added to the existing effect's remaining duration. Potency or other attributes are typically not modified.
*   **OVERWRITE:** The existing effect is completely removed and replaced by the new one. This effectively resets the duration and applies any new potency values.
*   **IGNORE:** The new effect application is discarded entirely, leaving the original effect to continue unmodified. This is useful for preventing weaker effects from overwriting stronger ones.

## Lifecycle & Ownership
-   **Creation:** Enum constants are instantiated once by the Java Virtual Machine (JVM) when the OverlapBehavior class is loaded. The static CODEC instance is also created at this time.
-   **Scope:** Application-wide. The enum constants are global, static, and persist for the entire lifetime of the server process.
-   **Destruction:** The constants are garbage collected only when the JVM shuts down. There is no manual lifecycle management.

## Internal State & Concurrency
-   **State:** Inherently immutable. The state of an enum constant cannot be changed after its creation.
-   **Thread Safety:** Fully thread-safe. As immutable singletons managed by the JVM, these constants can be safely accessed and compared from any thread without synchronization. The static final CODEC field is also thread-safe after its one-time initialization during class loading.

## API Surface
The primary API consists of the enum constants themselves, which are used for identity comparison and in switch statements.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EXTEND | OverlapBehavior | N/A | Policy to extend the duration of an existing effect. |
| OVERWRITE | OverlapBehavior | N/A | Policy to replace an existing effect with a new instance. |
| IGNORE | OverlapBehavior | N/A | Policy to discard a new effect if one already exists. |
| CODEC | EnumCodec | N/A | Static codec for serializing/deserializing this enum from data assets. |

## Integration Patterns

### Standard Usage
This enum is primarily consumed by the Entity Effect System after being loaded from an asset definition. Game logic then uses the configured value to apply the correct behavior.

```java
// Pseudo-code demonstrating consumption by the game logic
public void applyEffect(Entity target, Effect newEffect) {
    Effect existingEffect = target.findEffect(newEffect.getType());

    if (existingEffect != null) {
        // The behavior is read from the new effect's data-driven configuration
        OverlapBehavior behavior = newEffect.getConfig().getOverlapBehavior();

        switch (behavior) {
            case EXTEND:
                existingEffect.addDuration(newEffect.getDuration());
                break;
            case OVERWRITE:
                target.replaceEffect(existingEffect, newEffect);
                break;
            case IGNORE:
                // Do nothing
                break;
        }
    } else {
        target.addNewEffect(newEffect);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Reflection-based Instantiation:** Do not attempt to create new instances of OverlapBehavior using reflection. This violates the JVM's contract for enums and will lead to unpredictable system behavior and failed equality checks.
-   **String Comparison:** Avoid comparing the enum's name as a string. Always use direct object comparison (==) or the equals method for type safety and performance.

## Data Pipeline
The OverlapBehavior enum is a terminal point in the asset loading pipeline, transforming raw data into a type-safe, machine-readable policy.

> Flow:
> Effect Asset File (JSON) -> Asset Loading Service -> **EnumCodec** -> **OverlapBehavior instance** -> Effect Configuration Object -> Entity Effect System

