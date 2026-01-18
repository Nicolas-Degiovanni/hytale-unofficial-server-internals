---
description: Architectural reference for POIMarkerProvider
---

# POIMarkerProvider

**Package:** com.hypixel.hytale.server.core.universe.world.worldmap.markers
**Type:** Singleton

## Definition
```java
// Signature
public class POIMarkerProvider implements WorldMapManager.MarkerProvider {
```

## Architecture & Concepts
The POIMarkerProvider is a stateless, logic-only component responsible for supplying Point of Interest (POI) map markers to the world map system. It acts as a specialized data source within the broader `WorldMapManager` framework.

Its sole responsibility is to bridge the gap between the world's persistent POI data, held by the WorldMapManager, and the per-player visibility calculations, managed by the WorldMapTracker. This class does not own or cache any data; it merely retrieves a list of global POIs and submits them to a tracker for processing during a player-specific update cycle.

This provider is part of a strategy pattern, where the WorldMapManager iterates over multiple MarkerProvider implementations (e.g., for player markers, quest markers, or POI markers) to aggregate all visible map icons for a given client.

### Lifecycle & Ownership
- **Creation:** As a static singleton, the single INSTANCE is initialized by the Java ClassLoader when the POIMarkerProvider class is first referenced. It is not instantiated by any engine or game logic.
- **Scope:** Application-level. The singleton instance persists for the entire lifetime of the server process.
- **Destruction:** The instance is eligible for garbage collection only when the server process is shutting down and its ClassLoader is unloaded.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields and its behavior is determined entirely by the arguments passed to its `update` method.
- **Thread Safety:** The POIMarkerProvider is inherently thread-safe. The `update` method is a pure function relative to the class itself. However, callers must ensure that the `World` and `WorldMapTracker` objects passed as arguments are accessed in a thread-safe manner, as this provider will read from one and write to the other.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| update(world, config, tracker, ...) | void | O(N) | Iterates all N global POIs and submits each to the tracker. The tracker performs subsequent proximity checks. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by gameplay logic. It is designed to be registered with and managed by the WorldMapManager. The manager is responsible for invoking the `update` method at the appropriate time for each player.

A system-level consumer, such as the WorldMapManager, would use it as follows:

```java
// Hypothetical usage within WorldMapManager
// This is NOT for general gameplay code.
for (MarkerProvider provider : registeredProviders) {
    // The manager invokes the provider for a specific player's tracker
    provider.update(world, config, player.getWorldMapTracker(), ...);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not use reflection to create new instances. Always use the static `POIMarkerProvider.INSTANCE`.
- **Manual Invocation:** Do not call `INSTANCE.update()` from gameplay code. The world map system relies on a specific update order and context provided by the WorldMapManager. Manual calls will bypass this orchestration and can lead to inconsistent or duplicated map data on the client.

## Data Pipeline
The POIMarkerProvider is a single step in the pipeline that sends world map data to the client. It is responsible for sourcing POI data and forwarding it for filtering and network serialization.

> Flow:
> WorldMapManager (Global POI List) -> **POIMarkerProvider.update()** -> WorldMapTracker (Proximity Filtering) -> Network Packet -> Client Render

---

