---
description: Architectural reference for SpawnMarkerProvider
---

# SpawnMarkerProvider

**Package:** com.hypixel.hytale.server.core.universe.world.worldmap.markers
**Type:** Singleton

## Definition
```java
// Signature
public class SpawnMarkerProvider implements WorldMapManager.MarkerProvider {
    public static final SpawnMarkerProvider INSTANCE = new SpawnMarkerProvider();
    // private constructor
}
```

## Architecture & Concepts
The SpawnMarkerProvider is a concrete implementation of the **MarkerProvider** strategy interface. Its sole responsibility is to supply the world's designated spawn point as a visual marker on the player's world map.

This class operates as a stateless, data-providing component within the server's world map system. It is not invoked directly but is instead registered with and orchestrated by the WorldMapManager. During the map update cycle, the WorldMapManager iterates through all registered providers, including this one, to aggregate a complete set of map markers for a given player.

Architecturally, it serves as a specialized adapter, translating high-level world configuration data (the spawn location) into a specific, client-consumable data structure (a MapMarker packet) via the WorldMapTracker.

## Lifecycle & Ownership
- **Creation:** The singleton INSTANCE is instantiated by the Java ClassLoader when the SpawnMarkerProvider class is first referenced. Its creation is not tied to the server or world lifecycle but rather to the application's startup sequence.
- **Scope:** Application-level. The single instance persists for the entire lifetime of the server process.
- **Destruction:** The object is eligible for garbage collection only upon JVM shutdown. No explicit cleanup logic is required or implemented.

## Internal State & Concurrency
- **State:** The SpawnMarkerProvider is **stateless and immutable**. It contains no instance fields and its behavior is purely a function of the arguments passed to its update method. All required context, such as the world, player, and configuration, is provided externally at the time of invocation.
- **Thread Safety:** This class is inherently thread-safe. As it holds no state, concurrent calls to the update method from different threads will not interfere with each other. However, the objects passed as arguments, particularly World and WorldMapTracker, are not guaranteed to be thread-safe and must be handled according to the calling context's concurrency model, which is typically a single world-update thread.

## API Surface
The public contract is minimal, consisting of the singleton accessor and the interface method implementation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| INSTANCE | SpawnMarkerProvider | O(1) | Provides global access to the singleton instance. |
| update(...) | void | O(1) | Evaluates world configuration and conditionally provides a spawn marker to the WorldMapTracker. |

## Integration Patterns

### Standard Usage
This provider is not intended for direct invocation. The correct pattern is to register the singleton instance with the central WorldMapManager, which will then manage its lifecycle and invocation.

```java
// Hypothetical registration during server initialization
WorldMapManager mapManager = server.getWorldMapManager();
mapManager.registerProvider(SpawnMarkerProvider.INSTANCE);
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Manually calling `SpawnMarkerProvider.INSTANCE.update(...)` is a design violation. This bypasses the orchestration, caching, and update-throttling logic within the WorldMapManager, potentially leading to redundant network packets or incorrect state.
- **Reflection-based Instantiation:** Attempting to create new instances of SpawnMarkerProvider via reflection violates the singleton pattern and serves no practical purpose, as the class is stateless. Always use the static INSTANCE field.

## Data Pipeline
The SpawnMarkerProvider acts as a single step in the broader process of generating and sending world map data to the client.

> Flow:
> WorldMapManager Update Tick -> **SpawnMarkerProvider.update()** -> Reads WorldConfig & GameplayConfig -> Writes to WorldMapTracker -> WorldMapTracker serializes MapMarker packet -> Network Layer -> Client Render

---

