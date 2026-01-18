---
description: Architectural reference for PointKnockback
---

# PointKnockback

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server.combat
**Type:** Transient / Configuration Model

## Definition
```java
// Signature
public class PointKnockback extends Knockback {
```

## Architecture & Concepts
The PointKnockback class is a data-driven model representing a specific knockback behavior within the server-side combat system. It is not a service or manager, but rather a configuration object that encapsulates the parameters and logic for calculating a knockback vector originating from a specific point in space.

Its primary architectural function is to decouple combat design from engine code. Game designers can define complex physical responses to interactions—such as a weapon hit or spell impact—in external configuration files (likely JSON). The Hytale server loads these files at startup, using the static CODEC field to deserialize the data into immutable PointKnockback instances.

This class calculates a velocity vector based on the relative positions of a source and a target. It allows for sophisticated effects by including parameters for offsetting the knockback's origin point, rotating the final vector, and applying distinct vertical and horizontal forces. It is a fundamental building block for creating varied and predictable combat physics.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale **Codec** system during the server's asset loading phase. The static CODEC field acts as a factory, deserializing configuration data from game files into a PointKnockback object. Manual instantiation is a critical anti-pattern.
- **Scope:** An instance's lifetime is tied to the configuration from which it was loaded. These objects are typically held by a higher-level definition object, such as an ItemDefinition or AbilityDefinition, and persist for the entire server session. They are effectively immutable singletons for a given configuration.
- **Destruction:** Instances are marked for garbage collection when the server shuts down or when a live configuration reload discards the owning definition object. There is no manual destruction mechanism.

## Internal State & Concurrency
- **State:** The object's state is **effectively immutable** after its creation by the codec. All fields, such as velocityY, rotateY, and the inherited force, are set once during deserialization and are not designed to be modified at runtime. The calculateVector method is a pure function that does not mutate internal state.
- **Thread Safety:** This class is **thread-safe**. Due to its immutable nature, a single instance can be safely shared and accessed by multiple threads concurrently without locks or synchronization. This is critical in a multi-threaded server environment where combat calculations for different players may occur in parallel.

## API Surface
The public contract is minimal, focused entirely on its single computational responsibility.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| calculateVector(source, yaw, target) | Vector3d | O(1) | Calculates the final knockback velocity. This is a pure function that returns a new Vector3d instance and does not modify the object's state. |

## Integration Patterns

### Standard Usage
A high-level combat or ability system retrieves a pre-configured PointKnockback instance from a relevant game object definition. It then invokes calculateVector with the current positional data of the interacting entities to produce a velocity vector for the physics engine.

```java
// Assume 'combatEvent' holds contextual data for a server-side interaction
// and 'knockbackConfig' is a PointKnockback instance loaded from an item's definition.
PointKnockback knockbackConfig = combatEvent.getEffect().getKnockback();

Vector3d sourcePosition = combatEvent.getSourceEntity().getPosition();
float sourceYaw = combatEvent.getSourceEntity().getYaw();
Vector3d targetPosition = combatEvent.getTargetEntity().getPosition();

// Calculate the final velocity based on the configuration
Vector3d knockbackVelocity = knockbackConfig.calculateVector(sourcePosition, sourceYaw, targetPosition);

// Apply the calculated velocity to the target's physics component
combatEvent.getTargetEntity().getPhysicsComponent().applyVelocity(knockbackVelocity);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new PointKnockback()`. All instances must be defined in data files and loaded via the codec system. Bypassing this pattern breaks the data-driven design of the combat module and can lead to inconsistent behavior.
- **Runtime State Mutation:** Do not attempt to modify the public fields of a PointKnockback instance after it has been loaded. These objects are shared across the server; runtime mutation will cause unpredictable and widespread side effects. If a different knockback behavior is required, a new configuration file must be created.

## Data Pipeline
PointKnockback acts as a transformation node within the server's combat data flow, converting positional data into a physics instruction.

> Flow:
> Game Asset (JSON) -> Codec Deserializer -> **PointKnockback Instance** -> Combat System -> **PointKnockback.calculateVector()** -> Velocity Vector -> Physics Engine -> Entity Movement Update

