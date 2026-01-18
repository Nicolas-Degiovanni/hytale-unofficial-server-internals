---
description: Architectural reference for ParticleSpawnerGroup
---

# ParticleSpawnerGroup

**Package:** com.hypixel.hytale.server.core.asset.type.particle.config
**Type:** Configuration Model

## Definition
```java
// Signature
public class ParticleSpawnerGroup implements NetworkSerializable<com.hypixel.hytale.protocol.ParticleSpawnerGroup> {
```

## Architecture & Concepts

The ParticleSpawnerGroup class is a server-side data model that represents the configuration for a single group of particle emitters within a larger particle effect. It is not a live, in-world object but rather a static template or blueprint loaded from game assets. Its primary role is to hold all the defining properties of a particle spawner—such as its spawn rate, lifespan, velocity, and physical attractors—in a structured, in-memory format.

This class acts as a critical bridge between asset data and the network protocol. It is designed to be deserialized directly from a configuration file (e.g., JSON) via its static **CODEC** field. This `BuilderCodec` instance defines the mapping between serialized data keys and the object's fields, making it the canonical mechanism for instantiation.

Furthermore, by implementing the NetworkSerializable interface, this class defines the contract for its conversion into a `com.hypixel.hytale.protocol.ParticleSpawnerGroup` object. This allows the server to efficiently transmit detailed particle effect configurations to clients over the network.

## Lifecycle & Ownership

-   **Creation:** ParticleSpawnerGroup instances are not intended to be created manually. They are instantiated by the server's asset loading pipeline, which uses the static **CODEC** field to deserialize data from particle effect asset files. The protected default constructor exists exclusively for use by this codec mechanism.
-   **Scope:** An instance of this class typically has a singleton scope *per asset*. Once an asset file is loaded, the resulting ParticleSpawnerGroup object is cached by an asset management service and persists for the lifetime of the server process or until a hot-reload of assets is triggered. The same instance is reused every time that specific particle effect is requested.
-   **Destruction:** The object is eligible for garbage collection only when the asset manager that holds its reference is shut down or clears its cache.

## Internal State & Concurrency

-   **State:** The object's state is mutable during its initial construction by the `BuilderCodec`. After being fully deserialized and cached, it should be treated as **effectively immutable**. It is a read-only container for configuration data.
-   **Thread Safety:** This class is **not thread-safe for mutation**. It contains no internal locks or synchronization primitives. However, because it is designed as a read-only template after initialization, it is safe to be read concurrently by multiple game threads that need to spawn particle effects based on its configuration.

**WARNING:** Modifying a cached ParticleSpawnerGroup instance at runtime is a severe anti-pattern that will introduce global, unpredictable side effects, as the same instance is shared across the server.

## API Surface

The public API is dominated by simple getters. The most significant method is `toPacket`, which facilitates network serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.ParticleSpawnerGroup | O(N) | Serializes the configuration model into its network-transferable protocol representation. Complexity is linear based on N, the number of attractors. |

## Integration Patterns

### Standard Usage

A developer should never interact with this class's constructor. Instead, it is accessed as part of a larger asset. The server engine uses it to create live effects and synchronize configuration with clients.

```java
// 1. A ParticleSpawnerGroup is loaded from an asset file by the asset system.
// This step is handled automatically by the engine's AssetManager.
ParticleSpawnerGroup spawnerConfig = assetManager.getParticleConfig("my_effect.json").getSpawnerGroups()[0];

// 2. The server serializes the configuration into a packet to be sent to a client.
com.hypixel.hytale.protocol.ParticleSpawnerGroup packet = spawnerConfig.toPacket();
networkManager.sendToClient(player, packet);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new ParticleSpawnerGroup(...)`. The canonical source of truth for particle configuration is the on-disk asset files. Bypassing the asset pipeline by creating instances manually will lead to desynchronized or incorrect particle behaviors.
-   **Runtime Mutation:** Do not get a cached instance from an asset manager and modify its fields. This object is shared state. Any mutation will affect every subsequent particle effect spawned using this configuration, leading to difficult-to-diagnose bugs.

## Data Pipeline

The ParticleSpawnerGroup serves as an intermediate, structured representation of data as it flows from storage to the network.

> Flow:
> Asset File (e.g., JSON) -> Hytale Codec System -> **ParticleSpawnerGroup** (Server-Side In-Memory Model) -> toPacket() -> Protocol Packet -> Network Layer -> Client Rendering Engine

