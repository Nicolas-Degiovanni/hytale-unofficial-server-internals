---
description: Architectural reference for ClientEffectWorldSettings
---

# ClientEffectWorldSettings

**Package:** com.hypixel.hytale.server.core.universe.world
**Type:** Transient Data Object

## Definition
```java
// Signature
public class ClientEffectWorldSettings {
```

## Architecture & Concepts
ClientEffectWorldSettings is a server-side data container that encapsulates the configuration for client-side visual and post-processing effects within a world. It is not a service or manager, but rather a Plain Old Java Object (POJO) that represents a specific set of rendering parameters, such as sun position, bloom, and sunshafts.

The most critical architectural feature is the static **CODEC** field. This `BuilderCodec` defines a declarative mapping between serialized data keys (e.g., "SunHeightPercent") and the object's internal fields. This design indicates that the primary function of this class is to be a target for deserialization from persistent storage, such as world data files, and to act as a source for serialization into network packets.

It serves as the authoritative source of truth on the server for how a world should look and feel to a client. The server populates an instance of this class, and then uses its factory methods to generate network packets that synchronize this state with all players in the relevant world or zone.

## Lifecycle & Ownership
- **Creation:** Instances are created by the server's world-loading systems. The static CODEC is used to deserialize world configuration data directly into a new ClientEffectWorldSettings object. Manual instantiation may occur for default or dynamically generated worlds.
- **Scope:** The lifetime of an instance is tightly coupled to the lifetime of the server-side `World` object it configures. It persists in memory as long as the corresponding world is loaded.
- **Destruction:** The object is eligible for garbage collection when the world it belongs to is unloaded from the server. There are no manual cleanup or `close` methods.

## Internal State & Concurrency
- **State:** The internal state is entirely **mutable**. It is a collection of float values representing rendering parameters, each with a default value. The object is designed to be populated once upon creation and then read from.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization primitives. All mutations via its public setters must be performed from a single, controlled thread, typically the main server thread during world initialization. Concurrent modification from multiple threads will result in data corruption and unpredictable client-side rendering artifacts. Reading its state to create packets is safe as long as no concurrent writes are occurring.

## API Surface
The primary API surface consists of factory methods for creating network packets. Standard getters and setters are omitted for brevity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| createSunSettingsPacket() | UpdateSunSettings | O(1) | Constructs a network packet containing sun position and angle. |
| createPostFxSettingsPacket() | UpdatePostFxSettings | O(1) | Constructs a network packet containing post-processing effect values like bloom and sunshafts. |

## Integration Patterns

### Standard Usage
The canonical use case involves the server loading world data, using the instance to generate synchronization packets, and sending them to a client upon entering the world.

```java
// Server-side logic within a World or Zone manager
ClientEffectWorldSettings worldEffects = loadEffectsFromConfig("world_name.json");

// When a player joins the world, send them the settings
player.sendPacket(worldEffects.createSunSettingsPacket());
player.sendPacket(worldEffects.createPostFxSettingsPacket());
```

### Anti-Patterns (Do NOT do this)
- **Client-Side Instantiation:** Clients should never create instances of this class. It is a server-authoritative configuration object. Clients receive the *results* of this configuration via network packets.
- **Frequent Modification:** This object is not designed for high-frequency updates. Modifying its state and sending packets every frame would be highly inefficient. It is intended for "load-time" or infrequent, event-driven changes (e.g., a world event that changes the sun's color).
- **Unsynchronized Access:** Reading from and writing to the same instance from different threads without external locking is a severe race condition and is unsupported.

## Data Pipeline
This class acts as a transformation point, converting persisted configuration into transient network commands.

> Flow:
> World Configuration File -> Deserializer (using CODEC) -> **ClientEffectWorldSettings** (Server Memory) -> `create...Packet()` -> Network Packet -> Client Rendering Engine

