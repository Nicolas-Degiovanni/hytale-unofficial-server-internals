---
description: Architectural reference for GlobalSpawnProvider
---

# GlobalSpawnProvider

**Package:** com.hypixel.hytale.server.core.universe.world.spawn
**Type:** Strategy / Data Object

## Definition
```java
// Signature
public class GlobalSpawnProvider implements ISpawnProvider {
```

## Architecture & Concepts
The GlobalSpawnProvider is a concrete implementation of the ISpawnProvider strategy interface. Its role within the server architecture is to provide the simplest possible spawning logic: a single, static, and universal spawn point for all players, regardless of their identity or the state of the world.

This component is fundamentally data-driven. The presence of a static CODEC field indicates that it is designed to be deserialized directly from world configuration files. This allows world creators to define a fixed spawn point without modifying server code. In the context of the broader spawn system, this class serves as a foundational or fallback mechanism, contrasting with more complex providers that might calculate spawn points based on player teams, occupied regions, or dynamic game events.

## Lifecycle & Ownership
- **Creation:** An instance of GlobalSpawnProvider is created by the server's configuration loading system during world initialization. The static CODEC is invoked to deserialize the object from a data source, such as a world settings file. Programmatic instantiation is rare and generally discouraged.
- **Scope:** The object's lifecycle is tightly bound to its parent World instance. It persists in memory for the entire duration that the world is loaded on the server.
- **Destruction:** The object is eligible for garbage collection when the World it belongs to is unloaded. It does not manage any native resources and has no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The internal state is minimal, consisting of a single private field: the *spawnPoint* Transform. This state is established at creation time and is not intended to be modified thereafter. While the Transform object itself is mutable, the GlobalSpawnProvider class exposes no methods to alter it, making the provider effectively immutable from the perspective of its public API.
- **Thread Safety:** The class is **conditionally thread-safe**. All public methods are read-only operations on the internal *spawnPoint* field. As long as the *spawnPoint* is not modified after initialization, concurrent reads from multiple threads (e.g., different player login threads) are safe. The class uses no internal locking, relying on a "load once, read many" usage pattern.

**WARNING:** Direct modification of the internal Transform object via reflection or other means will break thread safety and lead to severe race conditions.

## API Surface
The public contract is focused entirely on retrieving spawn information.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSpawnPoint(World world, UUID uuid) | Transform | O(1) | Returns the single, static spawn point. The world and uuid parameters are ignored. |
| getSpawnPoints() | Transform[] | O(1) | Returns a new single-element array containing the spawn point. |
| isWithinSpawnDistance(Vector3d position, double distance) | boolean | O(1) | Performs a fast squared-distance check against the spawn point's position. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay logic. Instead, higher-level server systems, such as a WorldManager or PlayerLifecycleService, will hold a reference to the ISpawnProvider interface and use it to position newly-joining players.

```java
// Example from a hypothetical PlayerManager service
// The specific provider implementation is abstracted away.
ISpawnProvider spawnProvider = world.getSpawnProvider();

// The provider could be a GlobalSpawnProvider or a more complex type.
Transform spawnLocation = spawnProvider.getSpawnPoint(world, player.getUUID());

player.teleportTo(spawnLocation);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new GlobalSpawnProvider()`. The spawn system is designed to be data-driven. Spawn points must be defined in world configuration files to be loaded by the server, not hardcoded in plugins or game logic.
- **Runtime State Mutation:** Do not attempt to modify the spawn point after the world has loaded. This violates the "static" contract of this provider and can cause unpredictable behavior for players who join or respawn later.

## Data Pipeline
The GlobalSpawnProvider is not part of a real-time data processing pipeline. Rather, it is the *result* of a configuration loading pipeline at server startup.

> Flow:
> World Configuration File -> Server Codec Deserializer -> **GlobalSpawnProvider Instance** -> World Object -> Player Spawn Service

