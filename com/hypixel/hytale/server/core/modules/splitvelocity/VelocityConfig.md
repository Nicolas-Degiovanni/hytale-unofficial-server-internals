---
description: Architectural reference for VelocityConfig
---

# VelocityConfig

**Package:** com.hypixel.hytale.server.core.modules.splitvelocity
**Type:** Transient Data Model

## Definition
```java
// Signature
public class VelocityConfig implements NetworkSerializable<com.hypixel.hytale.protocol.VelocityConfig> {
```

## Architecture & Concepts
The VelocityConfig class is a server-side data model that defines the physics parameters for entity velocity, specifically concerning air and ground resistance. It is not a service or a manager, but rather a Plain Old Java Object (POJO) that acts as a configuration container.

Its primary architectural feature is its tight integration with the Hytale **Codec** system, exposed via the static public field CODEC. This design indicates that VelocityConfig instances are not intended to be manually constructed in code. Instead, they are deserialized from external data sources, such as JSON or HOCON configuration files that define game worlds, zones, or specific entity behaviors.

By implementing the NetworkSerializable interface, this class serves as a bridge between the server's internal configuration and the client-facing protocol. The toPacket method translates the server-side model into a network-optimized Data Transfer Object (DTO), ensuring that client-side physics predictions can be synchronized with the server's authoritative state.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale Codec framework during the deserialization of configuration assets. The public static CODEC field defines the schema, validation rules, and binding logic for this process.

-   **Scope:** The lifetime of a VelocityConfig object is bound to the scope of the configuration that loaded it. For example, if it is part of a biome's definition, it will persist in memory as long as that biome's configuration is active. It is not a global singleton and multiple, distinct instances can exist simultaneously.

-   **Destruction:** The object is marked for garbage collection when its owning configuration context is unloaded. It requires no explicit cleanup or resource management.

## Internal State & Concurrency
-   **State:** The internal state is mutable. The class holds several float and enum fields representing physics properties. While these fields have default values, their primary purpose is to be overwritten by the codec during deserialization. After initialization, the object is typically treated as immutable.

-   **Thread Safety:** **This class is not thread-safe.** It is a simple data container with no internal locking or synchronization. It is designed to be populated once by the codec system on a loading thread and then read by multiple systems, such as the main game loop or physics thread.

    **Warning:** Modifying a VelocityConfig instance after it has been loaded and shared with other systems will introduce severe race conditions and non-deterministic physics behavior. All configuration should be considered final after the initial load.

## API Surface
The public API is minimal, consisting primarily of accessors for the underlying physics properties and the network serialization contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getGroundResistance() | float | O(1) | Retrieves the base velocity multiplier applied to an entity on the ground. |
| getGroundResistanceMax() | float | O(1) | Retrieves the maximum resistance factor when velocity exceeds the threshold. |
| getAirResistance() | float | O(1) | Retrieves the base velocity multiplier applied to an entity in the air. |
| getAirResistanceMax() | float | O(1) | Retrieves the maximum air resistance factor when velocity exceeds the threshold. |
| getThreshold() | float | O(1) | Retrieves the velocity magnitude at which resistance begins to transition to its max value. |
| getStyle() | VelocityThresholdStyle | O(1) | Retrieves the interpolation style (e.g., Linear) for the resistance transition. |
| toPacket() | com.hypixel.hytale.protocol.VelocityConfig | O(1) | Creates a network-safe DTO of this configuration for client synchronization. |

## Integration Patterns

### Standard Usage
VelocityConfig is not retrieved from a service locator. It is typically held by a higher-level manager or component and passed to systems that require it, such as a physics engine.

```java
// Example from a hypothetical physics system that receives the config
public class EntityPhysicsSystem {

    public void applyResistance(Entity entity, VelocityConfig config) {
        if (entity.isOnGround()) {
            // Simplified logic; a real implementation would use threshold and style
            float resistance = config.getGroundResistance();
            entity.getVelocity().multiply(resistance);
        } else {
            float resistance = config.getAirResistance();
            entity.getVelocity().multiply(resistance);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new VelocityConfig()`. This bypasses the codec system and results in an object with default, likely incorrect, physics values. Configuration must always be loaded from an authoritative data source.

-   **Runtime Modification:** Do not get a reference to a VelocityConfig object and attempt to change its state after it has been loaded. This is not thread-safe and breaks the principle of configuration-as-code.

## Data Pipeline
The primary flow for this object is from a static configuration file on disk into an in-memory representation used by the live game server.

> Flow:
> Game Asset (e.g., world.json) -> Codec Deserializer -> **VelocityConfig Instance** -> Physics Engine -> Entity State Update -> **toPacket()** -> Network Layer -> Client Simulation

