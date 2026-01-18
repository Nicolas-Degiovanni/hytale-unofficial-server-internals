---
description: Architectural reference for ForceKnockback
---

# ForceKnockback

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server.combat
**Type:** Configuration Model

## Definition
```java
// Signature
public class ForceKnockback extends Knockback {
```

## Architecture & Concepts
The ForceKnockback class is a data-driven configuration model that defines a specific type of knockback behavior within the server-side combat system. Unlike positional knockback which calculates a vector based on the relative positions of the attacker and target, ForceKnockback applies a velocity in a *predetermined direction*.

This component is a concrete implementation within a polymorphic knockback system. The combat engine interacts with the abstract Knockback parent class, allowing game designers to swap out different knockback behaviors (e.g., ForceKnockback, PositionalKnockback) without changing game logic code.

Its primary architectural feature is the static **CODEC** field. This enables the entire object, including its direction and force, to be defined and deserialized from external data files (e.g., JSON or HOCON). This pattern decouples game mechanics from the source code, empowering designers to configure weapon and ability effects directly.

A critical post-processing step is embedded in the codec: `afterDecode(i -> i.direction.normalize())`. This ensures that the configured direction vector is always a unit vector, preventing unintended scaling issues when the force is applied.

### Lifecycle & Ownership
- **Creation:** Instances are exclusively created by the Hytale **Codec** engine during the server's asset loading phase. It is deserialized from a configuration block that specifies a knockback type of ForceKnockback.
- **Scope:** The object's lifetime is bound to its parent configuration object, such as a weapon definition or an ability effect. It persists in memory as long as that parent configuration is loaded.
- **Destruction:** The object is marked for garbage collection when its parent configuration is unloaded or no longer referenced, typically upon server shutdown or a hot-reload of game assets.

## Internal State & Concurrency
- **State:** ForceKnockback is stateful, containing a `direction` vector and an inherited `force` scalar. However, its state is considered **effectively immutable** after deserialization. The fields are populated once by the codec and are not designed to be modified at runtime.
- **Thread Safety:** The class is **thread-safe for read operations**. The primary method, calculateVector, is a pure function that does not modify internal state. It operates on clones of internal data, making it safe to be called concurrently by multiple threads in the game loop (e.g., for processing simultaneous combat events).

## API Surface
The public contract is minimal, focused entirely on the calculation logic inherited from its parent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| calculateVector(source, yaw, target) | Vector3d | O(1) | **Core Logic.** Calculates a final velocity vector. The configured direction is rotated by the source entity's yaw and scaled by the force. **WARNING:** This implementation completely ignores the source and target position vectors. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use. It is consumed polymorphically by systems that require a Knockback object from a configuration source.

```java
// Pseudo-code for a combat system
// 1. A weapon's configuration is loaded, which contains a Knockback object.
//    This object happens to be a ForceKnockback instance.
Knockback configuredKnockback = weaponConfig.getKnockback();

// 2. During combat, the system uses the object without knowing its concrete type.
Vector3d sourcePos = attacker.getPosition();
float sourceYaw = attacker.getYaw();
Vector3d targetPos = victim.getPosition();

Vector3d resultingVelocity = configuredKnockback.calculateVector(sourcePos, sourceYaw, targetPos);

// 3. The velocity is applied to the victim.
victim.getPhysics().applyVelocity(resultingVelocity);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ForceKnockback()`. Doing so bypasses the **CODEC** system, most critically skipping the `normalize()` operation on the direction vector. This will lead to incorrect and unpredictable knockback calculations.
- **Runtime State Mutation:** Do not get an instance of this object and modify its public `direction` or `force` fields at runtime. These objects may be shared as part of a cached configuration, and mutation will cause server-wide inconsistencies.

## Data Pipeline
The flow for this class is strictly from configuration data into the running server, where it is used for calculations.

> Flow:
> Weapon Config File (JSON/HOCON) -> Hytale Codec Engine -> **Deserialized ForceKnockback Instance** -> Combat Logic Service -> Physics Engine -> Entity Movement Update

