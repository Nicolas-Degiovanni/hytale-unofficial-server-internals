---
description: Architectural reference for IndividualSpawnProvider
---

# IndividualSpawnProvider

**Package:** com.hypixel.hytale.server.core.universe.world.spawn
**Type:** Transient

## Definition
```java
// Signature
public class IndividualSpawnProvider implements ISpawnProvider {
```

## Architecture & Concepts
The IndividualSpawnProvider is a concrete implementation of the ISpawnProvider strategy interface. Its primary function is to provide a deterministic, yet seemingly random, spawn location for a specific player from a predefined list of possible spawn points.

This provider is central to creating consistent player experiences. By hashing the player's unique identifier (UUID), it guarantees that a returning player will always be assigned the same spawn point from the list. This is crucial for worlds where players might establish bases or have a vested interest in their initial spawn location.

The presence of a static CODEC field indicates that this component is designed to be fully integrated with the engine's data persistence and serialization systems. World designers configure a list of Transform objects (position and orientation) in world data files, which are then deserialized into an instance of this class at runtime. It acts as a data-driven configuration object that encapsulates spawn logic.

## Lifecycle & Ownership
- **Creation:** Instances are primarily created by the engine's serialization layer when a world is loaded. The static CODEC is used to deserialize world data into a fully configured IndividualSpawnProvider object. It can also be instantiated programmatically by server-side game logic for dynamic or procedural spawn configurations.
- **Scope:** The lifetime of an IndividualSpawnProvider is bound to the lifetime of the object that owns it, typically a World or a world-level configuration manager. It holds no global state and is intended to be a self-contained component.
- **Destruction:** The object is eligible for garbage collection once its owning World or configuration is unloaded. It does not manage any native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The core state is the private Transform array named spawnPoints. This state is populated at construction time and is **effectively immutable**. There are no public methods to add, remove, or modify the spawn points after the object has been created.
- **Thread Safety:** This class is **thread-safe**. Its immutable-after-construction state guarantees that concurrent calls to getSpawnPoint from multiple player login threads will not result in race conditions or data corruption. No external locking is required when interacting with an instance of this class.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSpawnPoint(World world, UUID uuid) | Transform | O(1) | Calculates and returns a deterministic spawn point for a given player UUID. **Warning:** Throws ArithmeticException if the internal spawn point list is empty. |
| getSpawnPoints() | Transform[] | O(1) | Returns the underlying array of all possible spawn points. The returned array should be treated as read-only. |
| getFirstSpawnPoint() | Transform | O(1) | A convenience method to retrieve the first spawn point in the list, or null if the list is empty. |
| isWithinSpawnDistance(Vector3d pos, double dist) | boolean | O(N) | Checks if a given position is within a specified distance of *any* spawn point in the list. N is the number of spawn points. |

## Integration Patterns

### Standard Usage
The provider is typically retrieved from a world configuration and used by the player spawning system to determine the initial location for a player entity.

```java
// Retrieve the spawn provider associated with the current world
ISpawnProvider spawnProvider = world.getSpawnProvider();

// Get a deterministic spawn location for the connecting player
Transform playerSpawn = spawnProvider.getSpawnPoint(world, player.getUUID());

// Create and place the player entity at the determined transform
world.createPlayerEntity(player, playerSpawn);
```

### Anti-Patterns (Do NOT do this)
- **Empty Spawn List:** Never configure or instantiate this provider with an empty array of spawn points. Doing so will cause a fatal **ArithmeticException** (division by zero) when getSpawnPoint is called, crashing the player login thread.
- **Modifying Returned Array:** The array returned by getSpawnPoints is the direct internal reference. Modifying its contents will violate the immutability contract of the class and lead to unpredictable behavior for all subsequent spawn calculations. Treat the returned array as read-only.

## Data Pipeline
The IndividualSpawnProvider primarily acts as a data source for the spawn system, loaded from persistent storage.

> Flow:
> World Data File -> Engine Deserializer (using CODEC) -> **IndividualSpawnProvider Instance** -> World Spawn Logic -> Player Entity Transform

