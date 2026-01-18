---
description: Architectural reference for TargetEntityEffect
---

# TargetEntityEffect

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server.combat
**Type:** Data Object / Configuration

## Definition
```java
// Signature
public class TargetEntityEffect {
```

## Architecture & Concepts
TargetEntityEffect is a data-only configuration object that defines the properties of a status effect applied to an entity during an interaction, such as a combat event. It is not a service or a manager; it is a passive data structure that encapsulates a set of rules for how an effect should behave.

The class is designed to be deserialized from external configuration files, as evidenced by the static **CODEC** field. This `BuilderCodec` is the cornerstone of its integration into the engine's data-driven architecture. The server loads game definitions (e.g., for weapons, potions, or environmental hazards) from asset files, and this codec is responsible for instantiating and populating TargetEntityEffect objects from the raw data.

This model allows designers to define complex effect behaviors—such as duration, application chance, and stacking rules—in simple data files without modifying game code. The Combat System or other interaction modules then consume these configuration objects to execute game logic.

## Lifecycle & Ownership
- **Creation:** Instances are created by the Hytale **Codec** system during the server's asset loading phase. The static `CODEC` field is invoked by a higher-level configuration manager which reads data from a source like a JSON or HOCON file. Direct instantiation within game logic is strongly discouraged.
- **Scope:** The lifetime of a TargetEntityEffect instance is tied to its parent configuration object. For example, if it defines a poison effect for a specific sword, it will exist in memory as long as that sword's definition is loaded. It is not a global or session-scoped object.
- **Destruction:** The object is managed by the Java garbage collector. It is de-referenced and cleaned up when its parent configuration is unloaded, typically during a server shutdown or a dynamic asset reload. There are no explicit destruction methods.

## Internal State & Concurrency
- **State:** The state is mutable during the deserialization process via the `BuilderCodec`. After initialization, it is intended to be treated as a **read-only** data container. All fields are protected, discouraging external modification post-creation.
- **Thread Safety:** This class is **not inherently thread-safe**. It contains no locks or synchronization mechanisms. However, it is considered safe for concurrent reads under the assumption that it is never modified after the initial asset loading phase.

**WARNING:** Modifying the state of a TargetEntityEffect object at runtime after it has been loaded is a critical anti-pattern. This can lead to unpredictable and inconsistent behavior across different game threads that may be referencing the same shared configuration instance.

## API Surface
The public API is composed exclusively of accessors for reading the configured properties.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDuration() | float | O(1) | Returns the base duration of the effect in seconds. |
| getChance() | double | O(1) | Returns the probability (0.0 to 1.0) that the effect will be applied. |
| getEntityTypeDurationModifiers() | Object2DoubleMap<String> | O(1) | Returns a map of entity types to duration multipliers. |
| getOverlapBehavior() | OverlapBehavior | O(1) | Returns the rule for how to handle applying this effect to a target that already has it. |

## Integration Patterns

### Standard Usage
A system, such as the Combat System, retrieves this object from a parent configuration (e.g., a weapon definition) and uses its properties to apply a status effect to a target entity.

```java
// Conceptual example within a combat handler
void applyWeaponEffects(Entity attacker, Entity target, WeaponConfig weaponConfig) {
    // Assume weaponConfig holds a list of TargetEntityEffect objects
    for (TargetEntityEffect effectConfig : weaponConfig.getEffects()) {
        if (Random.nextDouble() <= effectConfig.getChance()) {
            // Logic to calculate final duration based on modifiers
            float finalDuration = calculateDuration(effectConfig, target.getType());

            // Apply the actual status effect to the target entity
            EffectService.applyEffect(target, effectConfig, finalDuration);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Procedural Instantiation:** Do not create instances of TargetEntityEffect programmatically in game logic. This defeats the purpose of the data-driven design. All effect definitions should originate from configuration files.
- **Runtime State Mutation:** Never modify the fields of a shared TargetEntityEffect instance after it has been loaded. This will cause race conditions and non-deterministic behavior. If a dynamic effect is needed, create a new, separate data object for it.

## Data Pipeline
The primary flow for this class is part of the server's configuration loading pipeline, not a real-time data processing pipeline.

> Flow:
> Game Asset File (e.g., `poison_sword.json`) -> Asset Loading Service -> **BuilderCodec** -> **TargetEntityEffect Instance** -> Stored in WeaponConfig -> Read by Combat System

