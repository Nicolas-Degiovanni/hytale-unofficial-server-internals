---
description: Architectural reference for DeployableAoeConfig
---

# DeployableAoeConfig

**Package:** com.hypixel.hytale.builtin.deployables.config
**Type:** Transient Data Object

## Definition
```java
// Signature
public class DeployableAoeConfig extends DeployableConfig {
```

## Architecture & Concepts

The DeployableAoeConfig class is a data-driven configuration object that defines the behavior of an Area-of-Effect (AoE) deployable entity. It is a concrete implementation of the Strategy pattern, where this class encapsulates the specific logic for AoE damage and effects, while the base DeployableConfig provides a common interface for all deployable types.

This class is not a service or a manager; it is a behavioral blueprint. Its primary role is to be deserialized from an asset file (e.g., JSON) via its static **CODEC** field. This allows game designers and modders to define complex AoE behaviors without writing Java code. The engine's Entity Component System (ECS) invokes the **tick** method on this object, which in turn queries the game world for targets and applies damage or status effects.

Architecturally, it acts as a bridge between the data layer (asset files) and the simulation layer (ECS). It directly interacts with several core server systems:
*   **Entity Component System:** Receives entity state via the **tick** method's parameters (Store, ArchetypeChunk) and queues state changes via the CommandBuffer.
*   **Spatial Query System:** Uses TargetUtil to perform efficient spatial lookups for entities within the defined AoE shape (Sphere or Cylinder).
*   **Damage & Effect Systems:** Dispatches damage events via DamageSystems and applies status effects through the EffectControllerComponent.

A single instance of DeployableAoeConfig is typically shared across all deployable entities of the same type, making its state management and thread safety critical considerations.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale **Codec** system during server startup or asset loading. The static **CODEC** field defines the deserialization logic from an asset file into a Java object. Manual instantiation is an anti-pattern.
-   **Scope:** The object is cached by the asset system and lives for the entire server session. A single instance is referenced by potentially thousands of in-game deployable entities.
-   **Destruction:** The object is eligible for garbage collection only when the corresponding asset is unloaded by the server's asset manager.

## Internal State & Concurrency
-   **State:** The object is designed to be effectively immutable after creation. All configuration fields like **startRadius** and **damageAmount** are set once during deserialization and are treated as read-only during the simulation. It contains one piece of mutable, lazily-initialized state: **processedDamageCause**, which caches a resolved asset reference.

-   **Thread Safety:** **WARNING:** This class is not strictly thread-safe, which presents a risk in a multi-threaded game server. The **tick** method itself is safe as it does not modify the object's state, operating instead on the per-entity data passed to it. However, the lazy initialization in **getDamageCause** contains a check-then-act race condition:
    ```java
    if (this.processedDamageCause == null) {
       this.processedDamageCause = ...; // Race condition here
    }
    ```
    If two threads call this method simultaneously on a new object, the asset lookup and assignment could occur twice. While the outcome is likely benign in this specific case, this pattern is hazardous. All reads of configuration data are safe.

## API Surface

The primary API contract is the **tick** method, which is invoked by the ECS. Other methods are protected helpers for internal logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(...) | void | O(N) | The main entry point called by the ECS each frame. Complexity is O(N) where N is the number of entities within the AoE. |
| getRadius(...) | float | O(1) | Calculates the current radius of the AoE, which can expand over time. |
| handleDetection(...) | void | O(N) | Queries for entities within the AoE shape and applies effects and damage to them. |
| attackTarget(...) | void | O(1) | Queues a damage command for a single target into the CommandBuffer. |

## Integration Patterns

### Standard Usage

Developers and designers do not interact with this class directly in Java. The standard pattern is to define a deployable's properties in an asset file, which the engine then loads to create a DeployableAoeConfig instance.

A conceptual asset file might look like this:
```json
{
  "parent": "hytale:deployable_base",
  "config": {
    "type": "DeployableAoeConfig",
    "Shape": "Cylinder",
    "StartRadius": 1.0,
    "EndRadius": 5.0,
    "Height": 3.0,
    "RadiusChangeTime": 2.5,
    "DamageInterval": 0.5,
    "DamageAmount": 10.0,
    "DamageCause": "hytale:fire_damage",
    "ApplyEffects": [ "hytale:burning" ],
    "AttackEnemies": true
  }
}
```
The engine's ECS is then solely responsible for calling the **tick** method on the resulting object for any active deployable entity using this configuration.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call **new DeployableAoeConfig()**. This creates an uninitialized object that will cause NullPointerExceptions and bypass the critical data-driven workflow.
-   **Runtime State Mutation:** Do not modify public or protected fields of a cached DeployableAoeConfig instance at runtime. This is not thread-safe and will affect **all** active deployables using that configuration, leading to unpredictable global side effects.
-   **Manual Invocation:** Do not call the **tick** method directly. It must be invoked by the Entity Component System to ensure proper state management, command buffering, and thread synchronization.

## Data Pipeline

The flow of data and control for this component is managed entirely by the server's core systems.

> Flow:
> Asset File -> Server Asset Loader -> **DeployableAoeConfig CODEC** -> In-Memory **DeployableAoeConfig** Instance -> DeployableComponent on an Entity -> ECS Tick Loop -> **tick()** method -> TargetUtil Query -> CommandBuffer -> Damage & Effect Systems

