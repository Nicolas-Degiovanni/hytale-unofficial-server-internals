---
description: Architectural reference for PortalMarkerProvider
---

# PortalMarkerProvider

**Package:** com.hypixel.hytale.builtin.portals.integrations
**Type:** Singleton

## Definition
```java
// Signature
public class PortalMarkerProvider implements WorldMapManager.MarkerProvider {
```

## Architecture & Concepts
The PortalMarkerProvider is a specialized implementation of the WorldMapManager.MarkerProvider interface. Its sole responsibility is to inject a map marker for the world's primary spawn point into the player's world map UI.

This class acts as a bridge between the server's spawn configuration and the client-side map rendering system. It is a passive component, registered with and controlled by the WorldMapManager, which orchestrates the collection of map markers from various sources.

The provider is tightly coupled to a specific server configuration: it only activates if the world is configured to use an IndividualSpawnProvider. This design ensures that the "Portal" or "Fragment Exit" marker only appears in game modes where a single, static spawn point is defined, preventing its display in dynamic or procedurally determined spawn scenarios.

## Lifecycle & Ownership
- **Creation:** The single instance, INSTANCE, is created by the Java Class Loader when the PortalMarkerProvider class is first referenced. This follows the initialization-on-demand holder idiom for singletons.
- **Scope:** Application-scoped. The singleton instance persists for the entire lifetime of the server process.
- **Destruction:** The instance is not explicitly destroyed. It is eligible for garbage collection only when the server shuts down and its Class Loader is unloaded.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no member fields and all operations are performed on method-scoped parameters. Its behavior is entirely deterministic based on the input provided by the WorldMapManager.
- **Thread Safety:** This class is inherently thread-safe. As a stateless object, it can be safely invoked by multiple threads concurrently without risk of race conditions or data corruption. Concurrency guarantees for the objects it interacts with, such as WorldMapTracker, are the responsibility of those respective systems.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| update(world, config, tracker, ...) | void | O(1) | Adds a spawn point marker to the tracker. This is the primary entry point, called by the WorldMapManager. |

## Integration Patterns

### Standard Usage
This class is not designed for direct invocation by gameplay code. It is a plugin for the WorldMapManager system. The engine automatically discovers and registers this provider. A hypothetical manual registration would look like this:

```java
// This class is not meant to be used directly.
// The WorldMapManager discovers and invokes providers automatically.
// The following is a conceptual example of its registration:

WorldMapManager mapManager = world.getService(WorldMapManager.class);
mapManager.registerProvider(PortalMarkerProvider.INSTANCE);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PortalMarkerProvider()`. This violates the singleton pattern and serves no purpose. Always use the static `PortalMarkerProvider.INSTANCE` field for access.
- **Manual Invocation:** Do not call the `update` method directly. The WorldMapManager is responsible for managing the update cycle and providing the correct context, including the player-specific WorldMapTracker. Bypassing the manager will lead to inconsistent or missing map data for clients.

## Data Pipeline
The PortalMarkerProvider participates in the flow of data from server-side world state to the client-side user interface.

> Flow:
> WorldMapManager Update Tick -> **PortalMarkerProvider.update()** -> Reads World Spawn Config -> Pushes MapMarker to WorldMapTracker -> Network Serialization -> Client UI Render

