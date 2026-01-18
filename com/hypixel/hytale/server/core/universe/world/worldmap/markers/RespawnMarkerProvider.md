---
description: Architectural reference for RespawnMarkerProvider
---

# RespawnMarkerProvider

**Package:** com.hypixel.hytale.server.core.universe.world.worldmap.markers
**Type:** Singleton

## Definition
```java
// Signature
public class RespawnMarkerProvider implements WorldMapManager.MarkerProvider {
```

## Architecture & Concepts
The RespawnMarkerProvider is a concrete implementation of the **MarkerProvider** strategy pattern. Its sole responsibility is to translate a specific piece of persistent player data—their designated respawn points—into visual markers for the in-game world map.

This class acts as a data adapter, bridging the Player Configuration system (specifically `PlayerRespawnPointData`) and the World Map rendering pipeline. It is a passive component, invoked by the `WorldMapManager` during its periodic update cycle for a given player. It does not store state or initiate its own logic; it simply responds to requests for data, reads from the player's profile, and transforms that data into the required `MapMarker` format.

This design decouples the `WorldMapManager` from the specifics of any single type of map marker, allowing new marker sources to be added to the system by simply creating new `MarkerProvider` implementations.

### Lifecycle & Ownership
- **Creation:** The `RespawnMarkerProvider` is a static singleton, exposed via the public final field `INSTANCE`. It is instantiated by the JVM during class loading and exists for the entire lifetime of the server.
- **Scope:** Application-scoped. A single instance serves all players and all worlds on the server.
- **Destruction:** The instance is not explicitly destroyed. It is eligible for garbage collection only when the server process shuts down and its class loader is unloaded.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no instance fields and does not cache any data between invocations of its `update` method. All required context is provided as arguments to the method.
- **Thread Safety:** The class is inherently thread-safe. As a stateless singleton, it can be safely accessed by multiple threads without risk of race conditions or data corruption. However, the objects it interacts with, such as the `WorldMapTracker`, are responsible for their own thread safety.

## API Surface
The public contract consists of a single method defined by the `WorldMapManager.MarkerProvider` interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| update(world, gameplayConfig, tracker, ...) | void | O(N) | Polls for player respawn points and registers them with the tracker. N is the number of respawn points, which is typically very small. |

## Integration Patterns

### Standard Usage
This provider is not intended for direct invocation by general game logic. It is designed to be registered with and managed by the `WorldMapManager`. The manager is responsible for calling the `update` method at the appropriate time in the server tick loop for each player.

```java
// Conceptual usage within the WorldMapManager
// A developer would not write this code, but this is how the system uses the class.

// During server initialization, the manager would discover or register the provider.
List<WorldMapManager.MarkerProvider> providers = new ArrayList<>();
providers.add(RespawnMarkerProvider.INSTANCE);

// During a player-specific update tick...
for (WorldMapManager.MarkerProvider provider : providers) {
    provider.update(currentPlayer.getWorld(), serverConfig, currentPlayer.getWorldMapTracker(), ...);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to create a new instance of `RespawnMarkerProvider` using reflection. Always use the static `INSTANCE` field to ensure the singleton pattern is respected.
- **Manual Invocation:** Calling the `update` method directly from other systems is incorrect. This bypasses the orchestration logic of the `WorldMapManager`, which can lead to redundant network packets, out-of-order updates, or incorrect context. All marker generation must flow through the `WorldMapManager`.

## Data Pipeline
The `RespawnMarkerProvider` functions as a single step in the world map data pipeline. It transforms server-side player data into a format that can be sent to the client for rendering.

> Flow:
> PlayerRespawnPointData (from Player Profile) -> **RespawnMarkerProvider.update()** -> WorldMapTracker.trySendMarker() -> Network Packet (MapMarker) -> Client World Map UI

