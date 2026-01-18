---
description: Architectural reference for DirectionalKnockback
---

# DirectionalKnockback

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server.combat
**Type:** Configuration Model

## Definition
```java
// Signature
public class DirectionalKnockback extends Knockback {
```

## Architecture & Concepts
DirectionalKnockback is a concrete implementation of the **Knockback** strategy. It represents a server-side combat mechanic where the resulting knockback vector is influenced by the attacker's orientation (yaw) in addition to the relative positions of the attacker and target.

This class is fundamentally a data-driven configuration object. It is not a service or a manager. Its primary role is to encapsulate the parameters for a specific type of knockback defined in external game asset files (e.g., JSON definitions for weapons or abilities).

The static **CODEC** field is the cornerstone of its design. It leverages Hytale's serialization framework to translate declarative configuration data into a live Java object. This allows game designers to define complex knockback behaviors without modifying engine code. It inherits base properties like *force* from the parent Knockback codec, demonstrating a compositional approach to configuration.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly via the constructor. They are materialized by the Hytale **Codec** system when parsing game configuration files. The engine reads a definition, finds the corresponding codec, and uses it to construct and populate a DirectionalKnockback instance.
- **Scope:** The lifetime of a DirectionalKnockback object is bound to the lifetime of its owning configuration object (e.g., a weapon definition). It persists as long as that asset is loaded in memory.
- **Destruction:** The object is eligible for garbage collection when its parent configuration asset is unloaded by the AssetManager. There is no manual destruction logic.

## Internal State & Concurrency
- **State:** The internal state consists of the floating-point values *relativeX*, *velocityY*, *relativeZ*, and the inherited *force*. This state is set once upon deserialization and is considered **effectively immutable**. There are no public or protected methods for mutating this state after construction.
- **Thread Safety:** This class is **inherently thread-safe**. The primary method, calculateVector, is a pure function that depends only on its arguments and the immutable internal state. It produces a new Vector3d instance without causing any side effects. It can be safely invoked from multiple threads, such as parallel physics or combat simulation ticks.

## API Surface
The public contract is minimal, focused entirely on executing the knockback calculation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| calculateVector(source, yaw, target) | Vector3d | O(1) | Computes the final knockback velocity. This is the core functional entry point of the class. It is a pure function with no side effects. |

## Integration Patterns

### Standard Usage
This class is not intended to be instantiated or managed directly. It is retrieved from a higher-level configuration object and used to calculate a physics vector.

```java
// Assume 'weaponData' is an object loaded from game configuration
// that contains a Knockback field.
Knockback knockbackEffect = weaponData.getKnockback();

// During a combat event, the system calculates the resulting velocity
Vector3d sourcePosition = attacker.getPosition();
float sourceYaw = attacker.getYaw();
Vector3d targetPosition = target.getPosition();

Vector3d resultingVelocity = knockbackEffect.calculateVector(sourcePosition, sourceYaw, targetPosition);

// The velocity is then applied to the target entity's physics component
target.getPhysicsComponent().applyImpulse(resultingVelocity);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new DirectionalKnockback()`. The object's state will be uninitialized and incorrect. All instances must be created via the engine's configuration loading and codec system.
- **State Mutation via Reflection:** Modifying the internal fields after creation is an unsupported and dangerous operation that breaks the immutability contract and can lead to unpredictable physics behavior across the server.

## Data Pipeline
The primary flow for this class is from a static configuration file into a usable in-memory object.

> Flow:
> Game Asset File (JSON) -> Engine Asset Loader -> **Hytale Codec Deserializer** -> **DirectionalKnockback Instance** -> Combat System Logic -> Physics Engine

