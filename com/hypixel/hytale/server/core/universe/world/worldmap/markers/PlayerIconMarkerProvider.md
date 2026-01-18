---
description: Architectural reference for PlayerIconMarkerProvider
---

# PlayerIconMarkerProvider

**Package:** com.hypixel.hytale.server.core.universe.world.worldmap.markers
**Type:** Singleton

## Definition
```java
// Signature
public class PlayerIconMarkerProvider implements WorldMapManager.MarkerProvider {
```

## Architecture & Concepts
The PlayerIconMarkerProvider is a stateless, server-side strategy component responsible for generating world map markers for player entities. It operates as a plugin for the WorldMapManager system, implementing the MarkerProvider interface. Its sole function is to translate the current state of player entities within a certain proximity into MapMarker data packets destined for a specific client.

This provider is not a standalone service; it is a discrete unit of logic invoked by a higher-level system, typically the WorldMapManager, during its update cycle. It encapsulates the rules for player visibility on the map, including distance culling based on chunk view radius and adherence to gameplay-defined filters (e.g., team visibility, spectator mode).

By isolating this logic, the engine can easily enable, disable, or replace the mechanism for displaying players on the map without altering the core map management or networking systems.

## Lifecycle & Ownership
- **Creation:** The PlayerIconMarkerProvider is a static singleton. A single instance, named INSTANCE, is created by the Java ClassLoader when the class is first referenced. Its lifecycle is not managed by any dependency injection framework or service locator.
- **Scope:** Application-scoped. The singleton instance persists for the entire lifetime of the server process.
- **Destruction:** The instance is eligible for garbage collection only when the server process is shutting down and its ClassLoader is unloaded. No manual cleanup is required.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no instance fields and all computations are performed exclusively on arguments passed into the update method. This design ensures that each invocation is idempotent and free from side effects related to previous calls.

- **Thread Safety:** The class is inherently **thread-safe** due to its stateless nature. However, it is designed to be called from the main server game loop thread. The objects it receives, such as World and WorldMapTracker, are not guaranteed to be thread-safe.

    **WARNING:** Calling the update method from a worker thread outside the primary game loop will almost certainly lead to race conditions and ConcurrentModificationExceptions when iterating over the world's player list. All invocations must be synchronized with the main world tick.

## API Surface
The public contract consists of a single method defined by the MarkerProvider interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| update(world, config, tracker, ...) | void | O(N) | Scans all N players in the world to generate markers for a single target player's map. Invoked by the WorldMapManager. |

## Integration Patterns

### Standard Usage
This provider is not intended for direct invocation by general game logic. It is automatically discovered or registered with the WorldMapManager, which then calls its update method for each player that has a world map open.

```java
// Hypothetical usage within a WorldMapManager update loop
// This code would exist inside the engine, not typical gameplay code.

WorldMapTracker playerTracker = getTrackerForPlayer(player);

// The manager iterates through all registered providers
for (MarkerProvider provider : registeredProviders) {
    // PlayerIconMarkerProvider.INSTANCE would be one of these providers
    provider.update(world, gameplayConfig, playerTracker, ...);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to create new instances via reflection. Always use the static PlayerIconMarkerProvider.INSTANCE field.
- **External Invocation:** Do not call the update method from your own game logic. This bypasses the throttling and state management handled by the WorldMapTracker and WorldMapManager, potentially causing network floods and inconsistent client-side map state.
- **Stateful Decoration:** Do not wrap this singleton in another class that adds state. The system relies on its stateless, globally-accessible nature.

## Data Pipeline
The PlayerIconMarkerProvider acts as a transformation step in the flow of data from the game world state to the client's network connection.

> Flow:
> World State (Player Positions) -> **PlayerIconMarkerProvider.update()** -> WorldMapTracker (Filtering & Throttling) -> MapMarker Packet Construction -> Network Layer -> Client UI

---

