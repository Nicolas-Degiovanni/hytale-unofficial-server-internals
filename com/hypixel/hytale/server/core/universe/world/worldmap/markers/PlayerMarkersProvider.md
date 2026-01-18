---
description: Architectural reference for PlayerMarkersProvider
---

# PlayerMarkersProvider

**Package:** com.hypixel.hytale.server.core.universe.world.worldmap.markers
**Type:** Singleton

## Definition
```java
// Signature
public class PlayerMarkersProvider implements WorldMapManager.MarkerProvider {
```

## Architecture & Concepts
The PlayerMarkersProvider is a stateless component within the server's World Map system. Its sole responsibility is to act as a data source, bridging a player's persistent, saved map markers with the real-time map update pipeline.

It implements the MarkerProvider interface, conforming to a contract that allows the WorldMapManager to poll various sources for map markers. This class specifically handles markers created and saved by the player, such as waypoints or points of interest. It does not generate or manage markers itself; it merely retrieves them from the player's profile (PlayerWorldData) and forwards them to the WorldMapTracker for processing and potential network dispatch.

This design decouples the persistence layer (player data) from the network and visibility logic (map tracking), allowing for a clean, single-responsibility architecture.

### Lifecycle & Ownership
- **Creation:** As a static singleton, the single INSTANCE is created by the JVM class loader when the PlayerMarkersProvider class is first referenced. It is not instantiated by any engine system during a bootstrap sequence.
- **Scope:** Application-level. The singleton instance persists for the entire lifetime of the server process.
- **Destruction:** The instance is eligible for garbage collection only upon server shutdown when its class loader is unloaded. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields and its behavior is determined entirely by the arguments passed to its methods. All operations are transient.
- **Thread Safety:** The class is inherently thread-safe due to its stateless design. However, the objects passed into the `update` method, particularly the WorldMapTracker, are not guaranteed to be thread-safe. It is critical that all interactions with this provider are orchestrated from the owning world's main tick thread to prevent race conditions in the underlying map and player data systems.

## API Surface
The public contract is defined by the MarkerProvider interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| update(world, config, tracker, ...) | void | O(N) | Polls for player-specific markers. N is the number of markers saved by the player in the given world. |

## Integration Patterns

### Standard Usage
This provider is not intended for direct invocation. It should be registered with the central WorldMapManager, which will then call the `update` method at the appropriate time during the map update cycle for a given player.

```java
// Example of how the WorldMapManager might use this provider
// This code would exist within the WorldMapManager, not in typical gameplay code.

// On initialization of the map system:
List<MarkerProvider> providers = new ArrayList<>();
providers.add(PlayerMarkersProvider.INSTANCE);
// ... add other providers

// During a player's map update tick:
for (MarkerProvider provider : providers) {
    provider.update(world, config, playerTracker, ...);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Manually calling `PlayerMarkersProvider.INSTANCE.update(...)` is incorrect. This bypasses the orchestration and caching layers of the WorldMapManager, potentially leading to redundant network packets or out-of-order updates. Always allow the manager to control the update lifecycle.
- **Instantiation:** The class is a singleton with a private constructor. Attempting to create a new instance is impossible and violates the design contract. Always use the static `INSTANCE` field.

## Data Pipeline
The PlayerMarkersProvider acts as a specific data source within the broader world map data flow. It is triggered by the system, not by an event.

> Flow:
> WorldMapManager Update Tick -> **PlayerMarkersProvider.update()** -> Read PlayerWorldData -> WorldMapTracker.trySendMarker() -> Network Packet Queue -> Client Render

