---
description: Architectural reference for RemovalBehavior
---

# RemovalBehavior

**Package:** com.hypixel.hytale.server.core.asset.type.entityeffect.config
**Type:** Enumeration

## Definition
```java
// Signature
public enum RemovalBehavior {
   COMPLETE,
   INFINITE,
   DURATION;
```

## Architecture & Concepts
The RemovalBehavior enumeration defines a fixed set of policies that govern the termination condition for an entity effect. It is a critical configuration component within the server-side asset system, dictating how and when an active effect, such as a buff or debuff, is removed from a game entity.

This is not merely a programmatic constant; it represents a core part of the game's data contract. The presence of a static EnumCodec field indicates that this type is designed to be serialized from and deserialized into asset files (e.g., JSON or HOCON definitions for spells or items). This allows game designers to specify effect lifecycles directly in data, without requiring code changes.

The available policies are:
-   **COMPLETE:** The effect persists until its internal logic or animation sequence concludes. It is self-terminating.
-   **INFINITE:** The effect persists indefinitely until it is explicitly removed by another game system, event, or player action.
-   **DURATION:** The effect is time-based and will be removed after a specified duration has elapsed.

## Lifecycle & Ownership
-   **Creation:** Instances are created automatically by the Java Virtual Machine (JVM) during class loading. Developers never instantiate this type directly.
-   **Scope:** Application-wide. The enum constants exist for the entire lifetime of the server process.
-   **Destruction:** Instances are garbage collected when the JVM shuts down.

## Internal State & Concurrency
-   **State:** Inherently immutable. The state of an enum constant is defined at compile time and cannot be altered.
-   **Thread Safety:** Fully thread-safe. As immutable singletons, enum constants can be safely accessed and passed between any threads without requiring synchronization mechanisms.

## API Surface
The primary API consists of the predefined constants and the serialization codec.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| COMPLETE | RemovalBehavior | O(1) | Policy for effects that terminate upon completion. |
| INFINITE | RemovalBehavior | O(1) | Policy for effects that persist until explicitly cleared. |
| DURATION | RemovalBehavior | O(1) | Policy for effects that terminate after a set time. |
| CODEC | EnumCodec | O(1) | Static codec for serializing/deserializing the enum from data sources. |

## Integration Patterns

### Standard Usage
This enum is typically used within a switch statement to implement logic based on the configured policy or assigned to a configuration object during asset loading.

```java
// Example of processing an effect based on its removal policy
EntityEffectConfig config = asset.getConfig();

switch (config.getRemovalBehavior()) {
    case DURATION:
        // Start a countdown timer for the effect
        scheduleRemovalTimer(entity, effect, config.getDuration());
        break;
    case INFINITE:
        // No automatic removal logic is scheduled
        break;
    case COMPLETE:
        // The effect's own logic will trigger its removal
        break;
}
```

### Anti-Patterns (Do NOT do this)
-   **String Comparison:** Never compare an enum instance to a string. This is inefficient, error-prone, and defeats the purpose of a type-safe enum.
    -   **BAD:** `if (behavior.name().equals("DURATION")) { ... }`
    -   **GOOD:** `if (behavior == RemovalBehavior.DURATION) { ... }`
-   **Null Assignment:** Avoid designing systems where a variable of this type can be null. An effect's removal policy should always be explicit. A null value indicates a critical failure in asset loading or configuration.

## Data Pipeline
RemovalBehavior is a terminal point in the asset loading pipeline, converting raw data into a type-safe configuration value used by the game logic.

> Flow:
> Entity Effect Asset (JSON) -> Hytale Codec Library -> **EnumCodec** -> **RemovalBehavior** instance -> EntityEffectConfig Object -> Server Game Logic

