---
description: Architectural reference for the Knockback abstract base class.
---

# Knockback

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server.combat
**Type:** Configuration Model / Abstract Base Class

## Definition
```java
// Signature
public abstract class Knockback {
```

## Architecture & Concepts
The Knockback class is an abstract base for all knockback effect configurations within the server's combat system. It is not a service or a manager, but rather a data-driven model that encapsulates both the properties (e.g., force, duration) and the behavioral logic (the vector calculation) of a specific type of knockback.

The central architectural pattern employed here is **polymorphic deserialization** via the Hytale Codec system. The static `CODEC` field, a `CodecMapCodec`, is the public entry point for the engine to load knockback definitions from configuration files. This allows game designers to define various knockback types (e.g., a simple push, a vertical launch, a vector-based impulse) in data files, which are then materialized into concrete subclasses of Knockback at runtime.

Each concrete implementation must provide an implementation for the `calculateVector` method, which defines the unique logic for that knockback type. This pattern decouples the high-level combat systems from the specific mathematical details of how a knockback vector is derived.

## Lifecycle & Ownership
-   **Creation:** Knockback objects are not instantiated directly using a constructor. They are created exclusively by the Hytale Codec engine during server initialization or when game configurations are loaded. The engine reads a data source (e.g., a JSON file defining a weapon's properties) and uses the `Knockback.CODEC` to deserialize the data into the appropriate concrete Knockback instance.
-   **Scope:** An instance of Knockback has a lifetime tied to its containing configuration. Typically, these objects are loaded once at server startup and persist for the entire server session, held by objects like weapon definitions or ability configurations.
-   **Destruction:** The object is marked for garbage collection when the server shuts down or performs a full configuration reload that discards the old configuration objects.

## Internal State & Concurrency
-   **State:** The internal state (force, duration, etc.) is populated once during deserialization. While the fields are not declared as final, the class is designed to be treated as **effectively immutable** post-creation. There are no public setters, enforcing a read-only contract after initialization.
-   **Thread Safety:** This class is **conditionally thread-safe**. Because its state is immutable after creation, a single instance can be safely read and used by multiple game logic threads simultaneously without locks. All concrete implementations of `calculateVector` are expected to be pure functions, producing output based only on their inputs and the object's immutable state, thus ensuring thread safety.

## API Surface
The public API is minimal, focusing on retrieving configuration values and executing the core vector calculation logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| calculateVector(sourcePos, sourceYaw, targetPos) | Vector3d | O(1) | **Abstract.** Calculates the final knockback vector. Must be implemented by all subclasses. |
| getForce() | float | O(1) | Returns the magnitude of the knockback force. |
| getDuration() | float | O(1) | Returns the duration over which the force is applied. A value of 0 implies an instantaneous impulse. |
| getVelocityType() | ChangeVelocityType | O(1) | Returns how the knockback velocity should be applied (e.g., Add, Set). |
| getVelocityConfig() | VelocityConfig | O(1) | Returns the associated detailed velocity configuration, if any. |

## Integration Patterns

### Standard Usage
A developer should never create a Knockback instance directly. Instead, it should be retrieved from a higher-level configuration object (e.g., an `AttackDefinition`). The primary interaction is to call `calculateVector` to get the resulting physics impulse.

```java
// Example from a hypothetical CombatSystem
void applyKnockback(Entity attacker, Entity target, AttackDefinition attack) {
    Knockback knockbackConfig = attack.getKnockback();
    if (knockbackConfig == null) {
        return;
    }

    Vector3d sourcePos = attacker.getPosition();
    float sourceYaw = attacker.getYaw();
    Vector3d targetPos = target.getPosition();

    // Calculate the specific knockback vector using the polymorphic method
    Vector3d impulse = knockbackConfig.calculateVector(sourcePos, sourceYaw, targetPos);

    // Apply the calculated impulse to the target's physics component
    target.getPhysics().applyImpulse(impulse);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ConcreteKnockbackType()`. This bypasses the critical data-driven configuration system and creates tightly-coupled, hardcoded logic. All knockback effects should be defined in external configuration files.
-   **State Mutation:** Do not use reflection or other means to modify the fields of a Knockback object after it has been created. This violates its immutability contract and can lead to unpredictable and inconsistent behavior across different systems that share the same configuration instance.
-   **Manual Calculation:** Avoid retrieving individual fields like `getForce` to manually re-implement the knockback logic. This defeats the purpose of the polymorphic `calculateVector` method and will fail to account for the specific behavior of different knockback types.

## Data Pipeline
The Knockback class exists as part of a configuration data pipeline, not a real-time event pipeline. It represents the materialized form of static game data.

> Flow:
> Game Asset (e.g., weapon.json) -> Server Configuration Loader -> Hytale Codec System -> **Knockback Instance** -> Combat System -> Physics Engine Update

