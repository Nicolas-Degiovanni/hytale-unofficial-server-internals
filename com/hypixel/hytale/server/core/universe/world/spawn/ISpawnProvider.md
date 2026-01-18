---
description: Architectural reference for ISpawnProvider
---

# ISpawnProvider

**Package:** com.hypixel.hytale.server.core.universe.world.spawn
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface ISpawnProvider {
```

## Architecture & Concepts
The ISpawnProvider interface defines a formal contract for determining the spawn location of an entity within a World. It serves as a critical abstraction layer, decoupling the server's core entity management systems from the specific rules that govern spawn point selection. This allows for different spawn behaviors (e.g., fixed world spawn, player-defined spawn points, team-based spawns) to be implemented and swapped without altering the calling systems.

Architecturally, this interface embodies the **Strategy Pattern**. A World object holds a concrete implementation of ISpawnProvider, delegating all spawn-related queries to it. This design is central to world configuration and gameplay customization.

The presence of a static CODEC field indicates that implementations of this interface are serializable. This is essential for persisting world state, as the chosen spawn strategy is saved as part of the world data itself. The transition from older, entity-based methods to a modern, component-based approach (using ComponentAccessor) highlights the engine's deep integration with an Entity Component System (ECS) architecture.

## Lifecycle & Ownership
As an interface, ISpawnProvider itself has no lifecycle. The following describes the lifecycle of a concrete *implementation* of this interface.

- **Creation:** An ISpawnProvider implementation is instantiated during the creation or loading of a World. The specific implementation is determined by the world's configuration data.
- **Scope:** The instance is scoped to its parent World. It lives for the entire duration that the World is loaded in memory on the server.
- **Destruction:** The instance is eligible for garbage collection when its parent World is unloaded and all references to it are released. There is no explicit destruction method defined in the contract.

## Internal State & Concurrency
- **State:** The interface contract is stateless. However, concrete implementations may be stateful. For example, an implementation might cache a list of valid spawn locations or read spawn coordinates from a configuration file, holding that data in memory.
- **Thread Safety:** The contract makes no guarantees about thread safety. Implementations are responsible for ensuring safe concurrent access if they manage mutable state that could be accessed from multiple server threads (e.g., the main game loop and a separate world generation thread). Callers should assume that implementations are not thread-safe unless explicitly documented otherwise.

## API Surface
The public contract focuses on retrieving spawn locations and checking proximity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSpawnPoint(Ref, ComponentAccessor) | Transform | O(N) | **Primary Method.** Retrieves the spawn point for an entity identified by its ECS reference. Complexity depends on the implementation's lookup algorithm. |
| getSpawnPoint(World, UUID) | Transform | O(N) | The core lookup method. Finds the spawn point for a given entity UUID within a specific world. |
| isWithinSpawnDistance(Vector3d, double) | boolean | O(1) | Checks if a given 3D position is within a specified distance of the spawn area. Crucial for gameplay mechanics like spawn protection. |
| getSpawnPoint(Entity) | Transform | O(N) | **Deprecated.** Legacy method for retrieving a spawn point via a direct entity object reference. |
| getSpawnPoints() | Transform[] | O(N) | **Deprecated.** Legacy method that returned a fixed array of possible spawn points. Its deprecation suggests a move to more dynamic spawn logic. |

## Integration Patterns

### Standard Usage
The ISpawnProvider should be retrieved from the World object it belongs to. It is used by higher-level systems, such as the player login handler, to correctly place new entities into the world.

```java
// A system responsible for player spawning
World playerWorld = ...;
UUID playerUuid = ...;

// Retrieve the world's configured spawn provider
ISpawnProvider spawnProvider = playerWorld.getSpawnProvider();

// Calculate the spawn location
Transform spawnTransform = spawnProvider.getSpawnPoint(playerWorld, playerUuid);

// Use the transform to position the entity
playerEntity.setTransform(spawnTransform);
```

### Anti-Patterns (Do NOT do this)
- **Implementation Casting:** Do not cast an ISpawnProvider instance to a concrete type. Your code should be agnostic to the specific spawn strategy being used by the world.
- **Result Caching:** Avoid caching the result of getSpawnPoint for an extended period. The correct spawn point can change dynamically (e.g., a player setting a new bed spawn). Re-query the provider when a definitive spawn location is needed.
- **Using Deprecated Methods:** Cease all use of getSpawnPoint(Entity) and getSpawnPoints(). These are scheduled for removal and rely on outdated engine patterns.

## Data Pipeline
The ISpawnProvider acts as a query service within the entity lifecycle pipeline. It does not process a continuous stream of data but rather responds to on-demand requests for spawn location data.

> Flow:
> Game Event (e.g., PlayerJoinsGame) -> PlayerManagerSystem -> Accesses **World** -> Retrieves **ISpawnProvider** -> getSpawnPoint(world, uuid) -> Returns **Transform** -> System places Entity in World

