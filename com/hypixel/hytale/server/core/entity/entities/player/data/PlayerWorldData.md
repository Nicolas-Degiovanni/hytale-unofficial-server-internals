---
description: Architectural reference for PlayerWorldData
---

# PlayerWorldData

**Package:** com.hypixel.hytale.server.core.entity.entities.player.data
**Type:** Data Container

## Definition
```java
// Signature
public final class PlayerWorldData {
```

## Architecture & Concepts
PlayerWorldData is a stateful data container that encapsulates all player-specific information relevant to a single game world. It acts as a dedicated data component, separating world-persistent state (like last position, respawn points, and map markers) from global player state or transient session state.

This class is a fundamental part of the server's player data persistence mechanism. Its primary architectural feature is the static **CODEC** field, a BuilderCodec instance that declaratively defines the serialization and deserialization contract for this object. This allows the engine to transparently save player world data to disk and load it back into memory without manual mapping logic.

Crucially, PlayerWorldData is not a standalone entity. It is designed to be owned and managed by a parent **PlayerConfigData** object. This ownership is reflected in the `playerConfigData.markChanged()` calls within its setters, implementing a *Dirty Flag* pattern. Any modification to a PlayerWorldData instance signals its parent container that the entire player configuration is now "dirty" and requires saving during the next persistence cycle.

## Lifecycle & Ownership
- **Creation:** An instance of PlayerWorldData is created under two circumstances:
    1.  **Deserialization:** The static CODEC instantiates the object when loading a player's saved data from a persistent store. The framework is then responsible for injecting the parent PlayerConfigData reference via `setPlayerConfigData`.
    2.  **First Join:** When a player joins a world for the very first time and has no existing data, a new PlayerWorldData instance is created by its parent PlayerConfigData.

- **Scope:** The object's lifetime is strictly bound to the player's session within a specific world. It is loaded into memory when the player entity is initialized in a world and persists until the entity is unloaded.

- **Destruction:** The object is eligible for garbage collection when its owning PlayerConfigData is dereferenced and collected, typically after a player disconnects or the server shuts down. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** The internal state is highly **mutable**. It serves as a live, in-memory cache of the player's progress and status within the current world. Fields are directly modified by game logic via public setters. The class also contains minor business logic, such as capping the number of stored death positions.

- **Thread Safety:** **WARNING:** This class is **not thread-safe**. It contains no synchronization primitives, locks, or volatile fields. All methods that mutate state (e.g., `setLastPosition`, `addLastDeath`) perform non-atomic writes.

    It is architecturally mandated that instances of PlayerWorldData are confined to a single thread, almost certainly the main server thread responsible for the player's entity ticks. Unsynchronized access from other threads will lead to data corruption, race conditions, and inconsistent persistence state.

## API Surface
The public API is designed for direct state manipulation by trusted server-side game systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setLastPosition(transform) | void | O(1) | Updates the player's last known position and marks the parent data as dirty for saving. |
| setLastMovementStates(states, save) | void | O(1) | Updates movement state. The `save` flag allows for in-memory updates without triggering a persistence change. |
| addLastDeath(id, transform, day) | void | O(1) | Adds a new death location, removing the oldest if the list exceeds the maximum size. Marks data as dirty. |
| removeLastDeath(markerId) | void | O(N) | Removes a death position by its unique marker ID. N is the number of death positions, capped at 5. |
| setPlayerConfigData(config) | void | O(1) | **Framework Internal.** Sets the back-reference to the owning parent object. Critical for the dirty flag mechanism. |

## Integration Patterns

### Standard Usage
This class should never be interacted with directly. It must be accessed through its owning PlayerConfigData, which is typically retrieved from the player entity itself.

```java
// Correctly access and modify PlayerWorldData through its owner
Player player = ...; // Obtain player entity
PlayerConfigData config = player.getConfigData();
PlayerWorldData worldData = config.getWorldData();

worldData.setLastPosition(player.getTransform());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new PlayerWorldData()`. This results in a disconnected object where the `playerConfigData` field is null. Any method call that attempts to mark the state as changed will throw a NullPointerException, and the data will never be saved.

- **Multi-threaded Access:** Do not read or write to a PlayerWorldData instance from any thread other than the one that owns the corresponding player entity. This will cause severe concurrency issues.

- **State Caching:** Do not hold a long-lived reference to a PlayerWorldData object. Always retrieve it from the parent PlayerConfigData to ensure you are working with the currently active instance.

## Data Pipeline
PlayerWorldData is a critical link in the player data persistence pipeline.

> **Save Flow:**
> Game Event -> Game Logic calls setter on **PlayerWorldData** -> `playerConfigData.markChanged()` is invoked -> Server Persistence System detects dirty flag -> **PlayerWorldData.CODEC** serializes object state -> Binary Data written to disk

> **Load Flow:**
> Player Joins Server -> Server Persistence System reads Binary Data from disk -> **PlayerWorldData.CODEC** deserializes data into a new **PlayerWorldData** instance -> Framework attaches instance to a PlayerConfigData object -> Game Logic reads initial state from the instance

