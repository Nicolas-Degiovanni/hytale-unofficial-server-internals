---
description: Architectural reference for BiomeDataSystem
---

# BiomeDataSystem

**Package:** com.hypixel.hytale.server.worldgen
**Type:** System Component

## Definition
```java
// Signature
public class BiomeDataSystem extends DelayedEntitySystem<EntityStore> {
```

## Architecture & Concepts

The BiomeDataSystem is a server-side system within the Entity Component System (ECS) framework responsible for discovering and updating a player's current biome and zone information. It acts as a crucial bridge between the **World Generation** subsystem and the **Player State** subsystem.

Its core function is to periodically poll the location of entities that represent players, query the world generator for detailed geographical context at that location, and commit the resulting zone and biome data to the player's state via the WorldMapTracker component.

Architecturally, this system is designed as a low-frequency poller. By extending DelayedEntitySystem with a one-second delay, it intentionally avoids executing on every game tick. This is a critical performance optimization, as a player's biome context does not need real-time, high-frequency updates. This design prevents the computationally expensive calls to the world generator from impacting server tick performance.

The system's operational scope is strictly defined by its ECS query, which targets entities possessing both a **Player** component and a **TransformComponent**.

## Lifecycle & Ownership

-   **Creation:** An instance of BiomeDataSystem is created by the server's central ECS System Registry during the server bootstrap sequence. It is not intended for manual instantiation by game logic.
-   **Scope:** The system is a long-lived, stateless service. A single instance persists for the entire server runtime, processing all relevant entities across the game world.
-   **Destruction:** The instance is destroyed and marked for garbage collection only when the server is shutting down or the world it belongs to is fully unloaded. The ECS framework manages its entire lifecycle.

## Internal State & Concurrency

-   **State:** The BiomeDataSystem is fundamentally **stateless**. It retains no data between invocations of its tick method. All required information is sourced directly from the components of the entity being processed or from external world services within the scope of a single tick.

-   **Thread Safety:** This system is **not thread-safe** for arbitrary concurrent access. However, it is designed to operate safely within the Hytale ECS framework, which guarantees that system ticks are executed in a synchronized, deterministic order for a given world. All state modifications are deferred through a CommandBuffer, ensuring that changes are applied safely at the end of the tick cycle.

    **WARNING:** Never invoke the tick method directly from an external thread. All interaction must be mediated by the ECS scheduler.

## API Surface

The primary public contract of this class is its registration and interaction with the ECS scheduler, not direct invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(...) | void | O(N) | Framework-invoked method. Executes the core logic for a single entity. Complexity is dependent on the world generator's lookup algorithm. |
| getQuery() | Query | O(1) | Framework-invoked method. Returns the component query that filters entities for this system. Defines the system's operational scope. |

## Integration Patterns

### Standard Usage

Developers do not interact with this system directly. Its operation is automatic. To make an entity eligible for processing by this system, ensure it is created with the required components.

```java
// Example: Creating an entity that this system will process
// This code would exist within another system or entity factory.

void createPlayerEntity(CommandBuffer<EntityStore> commands, Vector3d spawnPosition) {
    // The Archetype defines the components the entity will have.
    Archetype playerArchetype = Archetype.of(
        Player.getComponentType(),
        TransformComponent.getComponentType()
        // ... other player components
    );

    // By creating an entity with this archetype, the BiomeDataSystem's
    // query will automatically match it and begin processing it.
    Ref<EntityStore> playerRef = commands.createEntity(playerArchetype);

    // Initialize the components
    TransformComponent transform = new TransformComponent();
    transform.setPosition(spawnPosition);
    commands.setComponent(playerRef, transform);

    Player player = new Player();
    // ... initialize player state
    commands.setComponent(playerRef, player);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new BiomeDataSystem()`. The ECS framework is responsible for creating and managing system instances. Manually creating one will result in a non-functional object that is not registered with the game loop.
-   **Manual Invocation:** Never call the `tick` method directly. This bypasses the ECS scheduler, the crucial 1-second delay, and the thread-safe CommandBuffer mechanism, which can lead to race conditions, world state corruption, and severe performance degradation.
-   **Dependency Misconfiguration:** The system gracefully handles cases where the world generator is not a ChunkGenerator. However, for biome and zone discovery to function, the active world must be configured with a compatible ChunkGenerator implementation.

## Data Pipeline

The system orchestrates a clear, one-way flow of data from the game world into the player's state.

> Flow:
> Player Position (from TransformComponent) -> **BiomeDataSystem.tick()** -> World Seed & Position Query -> ChunkGenerator.getZoneBiomeResultAt() -> Raw Zone & Biome Data -> WorldMapTracker.ZoneDiscoveryInfo (DTO) -> CommandBuffer -> WorldMapTracker Component Update

