---
description: Architectural reference for ParticleCollision
---

# ParticleCollision

**Package:** com.hypixel.hytale.server.core.asset.type.particle.config
**Type:** Configuration Model

## Definition
```java
// Signature
public class ParticleCollision implements NetworkSerializable<com.hypixel.hytale.protocol.ParticleCollision> {
```

## Architecture & Concepts

The ParticleCollision class is a data-holding model that defines the physical collision behavior of a particle within a particle effect. It is not a service or manager, but rather a component of a larger particle asset definition.

Its primary architectural role is to serve as a strongly-typed, in-memory representation of configuration data deserialized from an asset file (e.g., a JSON or HOCON file). This is accomplished via the static **CODEC** field, which uses the engine's reflection-based `BuilderCodec` system to map configuration keys like *BlockType* and *Action* to the class's internal fields. This declarative approach decouples the on-disk asset format from the server's internal logic.

Furthermore, this class acts as a translation layer between the server's asset system and the client-facing network protocol. By implementing the `NetworkSerializable` interface, it provides a contract for converting its configuration state into a `com.hypixel.hytale.protocol.ParticleCollision` packet, a data transfer object (DTO) optimized for network transmission.

## Lifecycle & Ownership

-   **Creation:** Instances are created exclusively by the engine's `Codec` framework during the asset loading phase. The static `BuilderCodec` uses the protected no-arg constructor to instantiate the object and subsequently populates its fields from the asset data. Manual instantiation is strongly discouraged.
-   **Scope:** The lifetime of a ParticleCollision object is bound to its parent particle asset. It is loaded once when the server starts or when assets are reloaded, and it persists in memory for the duration of the server's runtime, typically held by a central asset management system.
-   **Destruction:** The object is marked for garbage collection when its parent particle asset is unloaded from memory, which usually occurs during a server shutdown.

## Internal State & Concurrency

-   **State:** The class is a mutable Plain Old Java Object (POJO). Its state is populated once during deserialization from an asset file. After this initial loading process, instances of this class should be treated as **effectively immutable**. Modifying its state at runtime is a critical anti-pattern that can lead to inconsistent particle behavior.
-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms. Concurrency is managed by the broader asset loading framework, which ensures that these configuration objects are safely published to other threads only after they are fully constructed. Post-initialization, they are intended for read-only access from any thread.

## API Surface

The public API is designed for read-only access to the configured collision properties and for network serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getParticleMapCollision() | ParticleCollisionBlockType | O(1) | Retrieves the configured block type that this particle can collide with. |
| getType() | ParticleCollisionAction | O(1) | Retrieves the action the particle should take upon a successful collision, such as expiring. |
| getParticleRotationInfluence() | ParticleRotationInfluence | O(1) | Retrieves the configured influence that a collision has on the particle's rotation. |
| toPacket() | com.hypixel.hytale.protocol.ParticleCollision | O(1) | **CRITICAL:** Translates this server-side configuration model into its network DTO equivalent for transmission to the client. |

## Integration Patterns

### Standard Usage

Developers do not typically interact with this class directly. Instead, it is accessed as a property of a higher-level particle effect configuration object. Game logic systems would read its properties to implement collision physics.

```java
// Hypothetical usage within a particle simulation system
ParticleEffectDefinition effectDef = assetManager.getEffect("hytale:fire_burst");
ParticleCollision collisionConfig = effectDef.getCollisionConfiguration();

// Logic inside the particle update loop
if (particle.collidesWithWorld()) {
    ParticleCollisionAction action = collisionConfig.getType();
    if (action == ParticleCollisionAction.Expire) {
        particle.destroy();
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new ParticleCollision()`. All configuration must be driven by asset files to ensure consistency and maintain the data-driven design of the engine.
-   **Runtime Modification:** Never modify the fields of a ParticleCollision object after it has been loaded by the asset manager. This will cause unpredictable behavior and potential desynchronization between the server's simulation and what clients are told to render.

## Data Pipeline

The ParticleCollision class is a key component in the pipeline that transforms a static asset file into live particle behavior on the client.

> Flow:
> Particle Asset File (e.g., JSON) -> `BuilderCodec` Deserializer -> **ParticleCollision** (In-Memory Server Model) -> `toPacket()` -> `com.hypixel.hytale.protocol.ParticleCollision` (Network DTO) -> Client Particle Renderer

