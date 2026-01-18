---
description: Architectural reference for PhysicsConfig
---

# PhysicsConfig

**Package:** com.hypixel.hytale.server.core.modules.projectile.config
**Type:** Strategy Interface

## Definition
```java
// Signature
public interface PhysicsConfig extends NetworkSerializable<com.hypixel.hytale.protocol.PhysicsConfig> {
```

## Architecture & Concepts
The PhysicsConfig interface defines a strategic contract for applying physical behaviors to server-side entities, primarily projectiles. It is a core component of Hytale's data-driven Entity Component System (ECS), allowing designers to define complex physics interactions in external configuration files rather than hard-coding them.

This interface acts as a polymorphic bridge between entity configuration data and the server's physics simulation loop. The static CODEC field, a CodecMapCodec, is the key to this system. It deserializes different implementations of PhysicsConfig based on a "Type" identifier in the source data (e.g., JSON or HOCON files). This enables various physics models—such as standard gravity, zero-gravity, or custom flight patterns—to be attached to entities dynamically.

As it implements NetworkSerializable, a representation of this configuration can be transmitted to clients, likely to facilitate client-side prediction and smoother rendering of projectile motion.

### Lifecycle & Ownership
- **Creation:** Instances are not created directly via a constructor in game logic. They are instantiated by the Hytale codec system during the server's asset loading phase. The CodecMapCodec reads an entity or projectile definition file and constructs the appropriate concrete PhysicsConfig object (e.g., GravityPhysicsConfig, HomingPhysicsConfig).
- **Scope:** PhysicsConfig objects are stateless, immutable configurations. They are loaded once at startup and persist for the entire server session, shared across all entities that reference them.
- **Destruction:** These objects are garbage collected when the server shuts down and the associated asset registries are cleared.

## Internal State & Concurrency
- **State:** Implementations of this interface are expected to be **immutable**. They hold configuration values (like a gravity constant) but their internal state must not change after creation. The `apply` method operates on external state passed into it, not its own.
- **Thread Safety:** Inherently **thread-safe**. Due to their immutable nature, a single PhysicsConfig instance can be safely accessed and used by multiple world simulation threads simultaneously without requiring locks or synchronization.

## API Surface
The public contract is focused on applying the defined physics and retrieving basic properties.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(...) | void | O(1) | The primary execution method. Modifies an entity's velocity and state based on the rules of the specific implementation. |
| getGravity() | double | O(1) | Returns the gravitational constant for this physics model. The default of 0.0 implies a zero-gravity environment unless overridden. |

## Integration Patterns

### Standard Usage
A developer does not typically interact with this interface directly. The server's physics or projectile update system retrieves the appropriate PhysicsConfig from an entity's components and invokes `apply` during each simulation tick.

```java
// Example from within a hypothetical ProjectileSystem update loop
void processProjectile(Holder<EntityStore> projectileHolder) {
    // Retrieve the config component from the entity
    PhysicsConfig config = projectileHolder.get(Components.PHYSICS_CONFIG);
    Vector3d velocity = projectileHolder.get(Components.VELOCITY);

    // The system calls apply; the developer does not
    if (config != null) {
        config.apply(projectileHolder, null, velocity, projectileHolder, true);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Creating a concrete implementation of PhysicsConfig that contains mutable state is a severe violation of the design. These objects are shared globally and must be immutable to prevent race conditions and unpredictable behavior.
- **Direct Invocation:** Manually calling the `apply` method outside of the server's core physics simulation loop can lead to desynchronization and non-deterministic outcomes. Physics should be handled exclusively by the authoritative server systems.

## Data Pipeline
The data for PhysicsConfig flows from configuration files into the running server, where it is then used to influence the state of game entities.

> **Configuration Flow:**
> Projectile Definition File (JSON/HOCON) -> Server Asset Loader -> **CodecMapCodec Deserializer** -> In-Memory PhysicsConfig Instance
>
> **Runtime Flow:**
> Server Tick -> ProjectileSystem Update -> Get Entity's PhysicsConfig -> **PhysicsConfig.apply()** -> Modify Entity Velocity Component -> Physics Simulation Result

