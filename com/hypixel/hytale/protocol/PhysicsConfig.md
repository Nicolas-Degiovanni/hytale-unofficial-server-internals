---
description: Architectural reference for PhysicsConfig
---

# PhysicsConfig

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Transfer Object (DTO)

## Definition
```java
// Signature
public class PhysicsConfig {
```

## Architecture & Concepts
The PhysicsConfig class is a fixed-layout data structure that encapsulates the physical simulation properties of a game entity. It is not a service or a manager, but rather a fundamental component of the Hytale network protocol, acting as a data contract between the client and server.

Its primary role is to provide a highly efficient, zero-overhead representation of physics parameters for network transmission. The class is designed for direct memory mapping onto a network buffer, as evidenced by its static `FIXED_BLOCK_SIZE` of 122 bytes and its `serialize` and `deserialize` methods that operate directly on Netty's ByteBuf.

Within the engine, an instance of PhysicsConfig is typically associated with an entity template or a specific entity instance. The physics simulation engine reads from this object to determine how an entity should behave under forces like gravity, collisions, and fluid dynamics.

## Lifecycle & Ownership
- **Creation:** PhysicsConfig instances are created on-demand. There are two primary instantiation paths:
    1.  **Deserialization:** The network layer creates instances by calling the static `deserialize` method when decoding an incoming game packet.
    2.  **Programmatic:** Game logic or content scripts create instances using the `new` keyword to define the physics for new entity types or to override default behaviors.
- **Scope:** The object's lifetime is tied to its owner. If it is part of a network packet, it is short-lived and eligible for garbage collection after the packet is processed. If it is held by a game entity, it lives as long as that entity.
- **Destruction:** There is no explicit destruction logic. Instances are managed by the Java Garbage Collector and are reclaimed when no longer referenced.

## Internal State & Concurrency
- **State:** The state of PhysicsConfig is entirely **mutable**. All fields are public and can be modified directly after instantiation. The class is a simple data container and performs no caching or complex state management.

- **Thread Safety:** This class is **not thread-safe**. Direct access to its public fields from multiple threads without external synchronization will lead to race conditions and unpredictable physics behavior.

    **WARNING:** Instances of PhysicsConfig are intended for use within a single-threaded context, such as the main game loop or a dedicated network thread. If an instance must be passed between threads, it should be treated as immutable or a defensive copy should be created using the `clone` method.

## API Surface
The public API is centered around serialization, validation, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static PhysicsConfig | O(1) | Constructs a new PhysicsConfig by reading 122 bytes from the given buffer at a specific offset. |
| serialize(buf) | void | O(1) | Writes the object's state as 122 bytes into the provided buffer. Throws if buffer is not writable. |
| validateStructure(buffer, offset) | static ValidationResult | O(1) | Performs a pre-check to ensure the buffer contains enough data for a valid read. Does not validate content. |
| clone() | PhysicsConfig | O(1) | Creates and returns a new PhysicsConfig instance with the same values as the original. |

## Integration Patterns

### Standard Usage
The most common pattern is to instantiate and configure a PhysicsConfig object when defining a new entity type. This object is then passed to the entity's constructor or a configuration setter.

```java
// Define physics for a bouncy projectile
PhysicsConfig bouncyBallConfig = new PhysicsConfig();
bouncyBallConfig.gravity = 9.8;
bouncyBallConfig.bounciness = 0.95;
bouncyBallConfig.bounceCount = 10;
bouncyBallConfig.density = 1.2;

// The engine would then associate this config with an entity
Entity projectile = new ProjectileEntity(bouncyBallConfig);
```

### Anti-Patterns (Do NOT do this)
- **Shared Mutable State:** Do not assign the same PhysicsConfig instance to multiple distinct entities and then modify it. This will cause all entities to share the new physics properties, which is rarely the intended behavior.

    ```java
    // BAD: Both entities will have their bounciness changed
    PhysicsConfig sharedConfig = new PhysicsConfig();
    Entity entityA = new Entity(sharedConfig);
    Entity entityB = new Entity(sharedConfig);
    sharedConfig.bounciness = 0.8; // Affects both A and B
    ```

- **Manual Serialization:** Avoid calling `serialize` or `deserialize` directly in game logic. These methods are low-level components of the network protocol layer. Manual invocation can easily corrupt the network data stream.

## Data Pipeline
PhysicsConfig is a critical payload in the data flow between the game state and the network layer.

**Outbound Flow (Serialization):**
> Game Entity State -> **PhysicsConfig** -> Packet Construction -> `serialize(ByteBuf)` -> Network Wire

**Inbound Flow (Deserialization):**
> Network Wire -> Netty ByteBuf -> Packet Decoder -> `deserialize(ByteBuf, offset)` -> **PhysicsConfig** -> Entity State Update

