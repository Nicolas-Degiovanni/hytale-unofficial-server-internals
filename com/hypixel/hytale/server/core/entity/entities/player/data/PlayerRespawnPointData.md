---
description: Architectural reference for PlayerRespawnPointData
---

# PlayerRespawnPointData

**Package:** com.hypixel.hytale.server.core.entity.entities.player.data
**Type:** Data Object

## Definition
```java
// Signature
public final class PlayerRespawnPointData {
```

## Architecture & Concepts
PlayerRespawnPointData is a fundamental data structure, not a service or manager. Its sole purpose is to encapsulate the state of a player's designated respawn location. It acts as a pure Data Transfer Object (DTO) or value object, holding the coordinates and a descriptive name for a single respawn point.

The most critical architectural feature of this class is the static final **CODEC** field. This field directly integrates the class with Hytale's serialization engine. It defines a strict data contract for how an instance is encoded to and decoded from a binary or text format. This mechanism is essential for:
1.  **Network Persistence:** Transmitting player data between the server and client.
2.  **Disk Persistence:** Saving and loading player state to world or character files.

By defining its own serialization logic via the BuilderCodec, PlayerRespawnPointData ensures data integrity and version compatibility across the entire game engine.

## Lifecycle & Ownership
-   **Creation:** Instances are created through two primary pathways:
    1.  **Programmatically:** Via the public constructor `new PlayerRespawnPointData(...)`, typically when game logic dictates a new respawn point is set (e.g., a player interacts with a bed).
    2.  **Deserialization:** By the engine's serialization framework using the static CODEC. The framework invokes the private no-argument constructor and populates the fields from the source data stream.

-   **Scope:** The lifetime of a PlayerRespawnPointData instance is strictly tied to its owning object, which is almost always a parent Player or PlayerData entity. It persists in memory only as long as the associated player's data is loaded.

-   **Destruction:** The object is managed by the Java garbage collector. It is eligible for destruction once the parent Player entity is unloaded and all references to the instance are released. There is no explicit `destroy` or `cleanup` method.

## Internal State & Concurrency
-   **State:** The internal state is **Mutable**. While the position fields are typically set only at creation, the `setName` method allows the respawn point's name to be modified post-instantiation. The state is composed of an integer-based block position, a double-precision exact respawn position, and a string name.

-   **Thread Safety:** This class is **Not Thread-Safe**. It contains no internal synchronization mechanisms such as locks or volatile keywords.

    **WARNING:** All read and write operations on a PlayerRespawnPointData instance must be performed from a single, controlling thread, typically the main server thread for the world or zone the player inhabits. Unsynchronized access from multiple threads will result in data corruption and undefined behavior.

## API Surface
The public API is minimal, focusing on data access and modification of the name field.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBlockPosition() | Vector3i | O(1) | Returns the discrete, integer-based coordinate of the respawn block. |
| getRespawnPosition() | Vector3d | O(1) | Returns the precise, floating-point coordinate where the player will appear. |
| getName() | String | O(1) | Returns the user-defined name for the respawn point. |
| setName(String name) | void | O(1) | Updates the name of the respawn point. |

## Integration Patterns

### Standard Usage
This object is not meant to be managed directly. It should be retrieved from a higher-level player management system, inspected, and potentially modified through a controlled API.

```java
// Correctly retrieve the data object from a parent entity
Player player = getPlayerById("some-uuid");
PlayerRespawnPointData respawnData = player.getRespawnData();

if (respawnData != null) {
    Vector3d location = respawnData.getRespawnPosition();
    world.teleportEntity(player, location);
}
```

### Anti-Patterns (Do NOT do this)
-   **State Desynchronization:** Modifying the object's state directly without notifying the owning system can cause the changes to be lost. If a `player.setRespawnName` method exists, it should be preferred over `player.getRespawnData().setName()`, as the former can trigger necessary persistence logic.
-   **Shared Mutable References:** Do not pass a reference to this object to other systems that may run on different threads. If another system needs this information, provide it with an immutable copy or the raw values.

## Data Pipeline
PlayerRespawnPointData is not a processing stage in a pipeline; it is the **payload**. Its primary role in data flow is to serve as the serializable representation of a player's respawn location.

> **Serialization Flow (Saving Player Data):**
> Player Entity State -> **PlayerRespawnPointData** instance -> `CODEC.encode()` -> Serialized Byte Stream -> Network Buffer / Disk

> **Deserialization Flow (Loading Player Data):**
> Network Buffer / Disk -> Serialized Byte Stream -> `CODEC.decode()` -> **PlayerRespawnPointData** instance -> Player Entity State

