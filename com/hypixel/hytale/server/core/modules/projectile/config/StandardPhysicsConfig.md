---
description: Architectural reference for StandardPhysicsConfig
---

# StandardPhysicsConfig

**Package:** com.hypixel.hytale.server.core.modules.projectile.config
**Type:** Configuration Data Object

## Definition
```java
// Signature
public class StandardPhysicsConfig implements PhysicsConfig {
```

## Architecture & Concepts
The StandardPhysicsConfig class is a declarative data container that defines the physical properties of a server-side projectile. It is not a live physics simulation component itself; rather, it serves as a blueprint or template that is deserialized from game asset files.

Its primary architectural role is to bridge the gap between static game data and the dynamic, runtime Entity-Component-System (ECS). When a projectile is spawned, this configuration object is used to instantiate and attach a corresponding **StandardPhysicsProvider** component to the projectile entity. This provider component then consumes the configuration values to execute the actual physics calculations during the game tick.

Furthermore, this class is responsible for serializing its state into a network packet via the `toPacket` method. This is critical for client-side prediction, as it ensures the client can simulate the projectile's trajectory using the exact same physical parameters as the server, reducing perceived latency.

The class heavily relies on the Hytale **Codec** system, specifically the `BuilderCodec`. The static `CODEC` field defines the schema for how this object is populated from a data source, mapping keys like "Gravity" and "Bounciness" to the internal fields of the class.

## Lifecycle & Ownership
-   **Creation:** Instances are almost exclusively created by the engine's Codec system during the asset loading phase. A projectile definition file (e.g., in JSON format) is parsed, and the `BuilderCodec` populates a new StandardPhysicsConfig instance. Manual instantiation via `new StandardPhysicsConfig()` is possible but generally incorrect, as it yields a default object. A static `DEFAULT` instance exists for fallback scenarios.

-   **Scope:** The object is transient and its lifetime is typically bound to the asset definition it represents. It is read-only after creation and can be shared and reused to configure thousands of projectile entities of the same type. It holds no entity-specific state.

-   **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
-   **State:** The object's state is mutable during its construction by the `BuilderCodec`. After this initial population, it should be treated as **effectively immutable**. Its fields represent a fixed set of physical constants. It does not cache any runtime data or maintain references to live game objects.

-   **Thread Safety:** This class is **not thread-safe**. The fields are public and not protected by locks. However, the standard usage pattern mitigates concurrency risks: the object is created and populated on a loading thread and subsequently only read from by the main game thread.

    **Warning:** Modifying the fields of a shared StandardPhysicsConfig instance from multiple threads will lead to race conditions and unpredictable physics behavior. It must only be read from after its initial creation.

## API Surface
The public API is focused on applying the configuration and serializing it for network transport.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(...) | void | O(1) | **Critical Method.** Instantiates and attaches a StandardPhysicsProvider component to the target entity, effectively applying this configuration to a live object. |
| toPacket() | com.hypixel.hytale.protocol.PhysicsConfig | O(1) | Serializes the configuration into a network-transferable packet object for client-side prediction. |
| getGravity() | double | O(1) | Retrieves the configured gravity value. |
| getBounciness() | double | O(1) | Retrieves the configured bounciness factor. |

## Integration Patterns

### Standard Usage
The intended use is to retrieve this configuration from a central registry or asset manager and use it to configure a newly created projectile entity.

```java
// Example: Spawning a new projectile
// 1. Obtain the configuration, likely from an asset definition.
StandardPhysicsConfig arrowConfig = AssetManager.getProjectileConfig("hytale:arrow");

// 2. Create the projectile entity.
Holder<EntityStore> projectileEntity = world.createEntity();

// 3. Apply the physics configuration to the entity.
arrowConfig.apply(projectileEntity, creatorRef, initialVelocity, componentAccessor, false);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new StandardPhysicsConfig()`. This bypasses the data-driven `CODEC` system and creates an object with default, likely incorrect, values. Always load configurations from their asset definitions.

-   **Post-Creation Modification:** Do not modify the fields of a configuration object after it has been loaded. These objects are often shared and reused for multiple entity spawns. Modifying a shared instance will affect all subsequent projectiles and is a source of severe, hard-to-debug bugs.

    ```java
    // BAD: Do not do this.
    StandardPhysicsConfig config = AssetManager.getProjectileConfig("hytale:arrow");
    config.gravity = -50.0; // This will affect ALL future arrows!
    ```

## Data Pipeline
The StandardPhysicsConfig acts as a key transformation point, converting static data into runtime behavior and network packets.

> **Server-Side Flow:**
> Game Asset File (e.g., JSON) -> Engine `Codec` System -> **StandardPhysicsConfig Instance** -> `apply()` -> StandardPhysicsProvider Component on Entity -> Physics System Tick

> **Client-Side Synchronization Flow:**
> **StandardPhysicsConfig Instance** (on Server) -> `toPacket()` -> Network Packet -> Client -> Client-Side Entity with matching physics properties for prediction


