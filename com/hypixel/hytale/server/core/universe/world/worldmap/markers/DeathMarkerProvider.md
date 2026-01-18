---
description: Architectural reference for DeathMarkerProvider
---

# DeathMarkerProvider

**Package:** com.hypixel.hytale.server.core.universe.world.worldmap.markers
**Type:** Singleton

## Definition
```java
// Signature
public class DeathMarkerProvider implements WorldMapManager.MarkerProvider {
```

## Architecture & Concepts
The DeathMarkerProvider is a specialized, stateless component that implements the WorldMapManager.MarkerProvider interface. Its sole responsibility is to supply world map markers corresponding to a player's previous death locations.

This class functions as a data provider within the larger World Map system. The WorldMapManager orchestrates the collection of all map markers by invoking registered providers like this one. The DeathMarkerProvider acts as a bridge, translating persistent player-specific data (PlayerDeathPositionData) into transient, network-serializable MapMarker packets destined for the client. It decouples the core map system from the specific logic of how and when death markers should be displayed.

The provider's logic is conditional, gated by the server's gameplay configuration. It will only generate markers if the WorldMapConfig.isDisplayDeathMarker setting is enabled, allowing server operators to toggle this feature globally.

### Lifecycle & Ownership
-   **Creation:** The single instance is created statically at class-loading time and exposed via the public static final field INSTANCE. This follows the Eager Initialization Singleton pattern.
-   **Scope:** The instance is global and persists for the entire lifetime of the server's Java Virtual Machine.
-   **Destruction:** The object is not explicitly destroyed. It is garbage collected when the server process terminates. No cleanup logic is required due to its stateless nature.

## Internal State & Concurrency
-   **State:** This class is **stateless**. It contains no instance fields and does not cache any data between invocations. All necessary information is provided as arguments to the update method.
-   **Thread Safety:** The class is inherently **thread-safe**. As a stateless singleton, it can be safely invoked from any thread without risk of race conditions or data corruption within the provider itself.

    **Warning:** While the provider is thread-safe, the objects passed to it, such as WorldMapTracker and PlayerDeathPositionData, may have their own concurrency constraints. The caller is responsible for ensuring that calls into the update method are synchronized with the game's main thread or an appropriate world thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| update(world, config, tracker, ...) | void | O(N) | The primary contract method. Iterates through a player's N death positions and sends a marker for each. This is the sole entry point for the class's logic. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by feature developers. It is automatically managed by the WorldMapManager, which calls the update method during its own processing cycle for a specific player. The primary integration is registering this provider with the manager.

```java
// Example of how the WorldMapManager might use this provider
// This code is conceptual and does not exist in DeathMarkerProvider.

// In WorldMapManager:
List<MarkerProvider> providers = List.of(DeathMarkerProvider.INSTANCE, ...);

// During player map update:
for (MarkerProvider provider : providers) {
    provider.update(world, config, tracker, ...);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The constructor is private. Do not attempt to create new instances via reflection. Always use the static DeathMarkerProvider.INSTANCE field.
-   **Manual Invocation:** Do not call the update method directly. This bypasses the orchestration and caching logic of the WorldMapManager and can lead to redundant network packets or desynchronized map state. Always let the WorldMapManager manage the provider's lifecycle.

## Data Pipeline
The provider transforms persisted player data into network packets for client-side rendering.

> Flow:
> Player Death Event -> **PlayerDeathPositionData** (persisted) -> WorldMapManager Update Cycle -> **DeathMarkerProvider.update()** -> WorldMapTracker.trySendMarker() -> **MapMarker Packet** -> Client Network Layer -> Client World Map UI

