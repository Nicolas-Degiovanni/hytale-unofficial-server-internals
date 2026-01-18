---
description: Architectural reference for WorldReturnPoint
---

# WorldReturnPoint

**Package:** com.hypixel.hytale.builtin.instances.config
**Type:** Data Structure / Transient

## Definition
```java
// Signature
public class WorldReturnPoint {
```

## Architecture & Concepts

The WorldReturnPoint class is a fundamental data structure, not an active service. It serves as a configuration object that encapsulates all the necessary information to relocate a player to a specific point in a specific world. Its primary role is to act as a data contract for systems dealing with player spawning, session management, and world configuration.

The most critical architectural feature is the static **CODEC** field. This `BuilderCodec` instance defines the canonical serialization and deserialization format for a WorldReturnPoint. This allows the engine to reliably read this data from world configuration files, player data, or network streams and instantiate it into a usable in-memory object. The presence of this codec signals that WorldReturnPoint is a key component of the engine's data persistence and networking layers.

It is designed to be a simple, mutable container. Higher-level services, such as a `PlayerSessionManager` or `WorldService`, are responsible for interpreting this data and executing the logic to move a player.

### Lifecycle & Ownership

-   **Creation:** WorldReturnPoint instances are created in two primary ways:
    1.  **Deserialization (Most Common):** The engine's codec system instantiates the object when loading data from a persistent source, such as a world configuration file, using the static `CODEC`.
    2.  **Programmatic Instantiation:** Game logic can create new instances directly using the constructor, for example, when a player sets a new home location or an administrator defines a new spawn point via a command.

-   **Scope:** The lifetime of a WorldReturnPoint is bound to its owner. It is a value object with no global state. If it is part of a world's configuration, it persists as long as that configuration is loaded in memory. If created for a one-off teleport operation, it is short-lived and eligible for garbage collection once the operation completes.

-   **Destruction:** Cleanup is managed by the standard Java garbage collector. There are no native resources or explicit destruction methods.

## Internal State & Concurrency

-   **State:** The class is fully **mutable**. All internal fields (world, returnPoint, returnOnReconnect) are exposed via public setters. This design allows for easy modification of return points by game systems after they have been created or loaded. It does not cache any data.

-   **Thread Safety:** This class is **not thread-safe**. Its mutable nature makes it unsafe for concurrent modification without external synchronization.

    **Warning:** If an instance of WorldReturnPoint is shared between multiple threads, the calling code is responsible for implementing appropriate locking mechanisms to prevent race conditions and ensure data consistency. Modifying a shared instance without a lock can lead to unpredictable behavior.

## API Surface

The public API consists primarily of constructors and standard accessors/mutators for its data fields.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WorldReturnPoint() | constructor | O(1) | Creates an empty instance, primarily for use by the codec system. |
| WorldReturnPoint(world, point, reconnect) | constructor | O(1) | Creates a fully configured instance with initial values. |
| getWorld() / setWorld(uuid) | UUID | O(1) | Accesses or mutates the target world's unique identifier. |
| getReturnPoint() / setReturnPoint(transform) | Transform | O(1) | Accesses or mutates the target location and orientation. |
| isReturnOnReconnect() / setReturnOnReconnect(bool) | boolean | O(1) | Accesses or mutates the flag for reconnect behavior. |
| clone() | WorldReturnPoint | O(1) | Creates a new WorldReturnPoint instance with the same values. |

## Integration Patterns

### Standard Usage

The most common pattern is for a service to retrieve an instance from a configuration source and use it as a data payload for another system.

```java
// Example: A world service loading its default spawn point
// The codec system would handle the instantiation behind the scenes.
WorldConfig config = world.getConfig();
WorldReturnPoint spawn = config.getDefaultSpawnPoint();

// The spawn object is then passed to another service
PlayerTeleportService.teleport(player, spawn);
```

### Anti-Patterns (Do NOT do this)

-   **Concurrent Modification:** Do not read and write to the same WorldReturnPoint instance from different threads without external locks. The object provides no internal guarantees of atomicity.

-   **Live Configuration Edits:** Avoid modifying a WorldReturnPoint instance that is part of a globally accessible configuration object without a proper commit or notification system. Other systems may read the inconsistent, partially-modified state. Instead, clone the object, modify the clone, and then atomically swap it into the configuration.

    ```java
    // BAD: Modifying a shared object directly
    WorldReturnPoint sharedSpawn = globalConfig.getSpawn();
    sharedSpawn.setWorld(newWorldUUID); // Another thread might read the old transform with the new UUID
    sharedSpawn.setReturnPoint(newTransform);

    // GOOD: Using the clone-and-swap pattern
    WorldReturnPoint newSpawn = globalConfig.getSpawn().clone();
    newSpawn.setWorld(newWorldUUID);
    newSpawn.setReturnPoint(newTransform);
    globalConfig.setSpawn(newSpawn); // Atomically update the reference
    ```

## Data Pipeline

WorldReturnPoint is not a processing stage in a pipeline; it is the **data** that flows through it.

> **Configuration Loading Flow:**
> World Data on Disk -> Engine File I/O -> Codec Deserializer -> **WorldReturnPoint Instance** -> World Configuration Service

> **Player Spawn Flow:**
> Player Join Event -> Session Manager -> (retrieves) **WorldReturnPoint** -> Player Teleportation Service -> Network Packet to Client

