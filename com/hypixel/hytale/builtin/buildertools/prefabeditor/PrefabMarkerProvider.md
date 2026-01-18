---
description: Architectural reference for PrefabMarkerProvider
---

# PrefabMarkerProvider

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor
**Type:** Singleton

## Definition
```java
// Signature
public class PrefabMarkerProvider implements WorldMapManager.MarkerProvider {
```

## Architecture & Concepts
The PrefabMarkerProvider is a specialized implementation of the MarkerProvider interface. It serves as a data bridge, translating the in-memory state of a player's prefab editing session into visual markers on the world map.

This class is not a standalone service but a **pluggable component** for the core WorldMapManager system. The WorldMapManager is responsible for periodically polling all registered providers for map markers relevant to a specific player's view. The PrefabMarkerProvider's sole function is to respond to these polls by inspecting the active PrefabEditSession for a given player and generating the appropriate markers.

This design decouples the Builder Tools feature set from the core map system, allowing for modular and maintainable code. The map system does not need to know what a "prefab" is; it only needs to process the generic MapMarker objects that this provider supplies.

### Lifecycle & Ownership
-   **Creation:** The PrefabMarkerProvider is a static singleton, instantiated eagerly by the JVM during class loading via its public static final INSTANCE field. It is not managed by a dependency injection framework.
-   **Scope:** Its lifecycle is tied directly to the server process. It exists from the moment its class is loaded until the server shuts down.
-   **Destruction:** The object is never explicitly destroyed. It is eligible for garbage collection only when the server process terminates.

## Internal State & Concurrency
-   **State:** This class is **stateless**. It contains no instance fields and does not cache any data between invocations. All necessary information is retrieved from external systems (PrefabEditSessionManager, WorldMapTracker) during the execution of the update method. This statelessness makes its behavior highly predictable.
-   **Thread Safety:** The class is inherently thread-safe due to its stateless design. However, the responsibility for safe invocation lies with the caller, the WorldMapManager. The engine guarantees that the update method is called from a safe context, typically a world-specific update thread, preventing race conditions with the game state it accesses.

## API Surface
The public contract is defined by the MarkerProvider interface and the static INSTANCE field used to access the singleton.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| update(...) | void | O(N) | Called by the WorldMapManager to generate markers. N is the number of loaded prefabs in the target player's session. |
| INSTANCE | PrefabMarkerProvider | O(1) | Provides global, static access to the singleton instance. |

## Integration Patterns
This component is designed to be registered with a central system and should not be invoked directly.

### Standard Usage
The singleton INSTANCE is registered with the WorldMapManager, typically during plugin initialization. The WorldMapManager then manages its lifecycle and invocation.

```java
// Example from a hypothetical plugin entry point
WorldMapManager mapManager = server.getWorldMapManager();

// The provider is registered once. The system handles the rest.
mapManager.registerProvider(PrefabMarkerProvider.INSTANCE);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new PrefabMarkerProvider()`. This violates the singleton pattern and serves no purpose, as the system is designed to work with the globally accessible INSTANCE.
-   **Manual Invocation:** Do not call the `update` method directly. This bypasses the orchestration and thread-safety guarantees provided by the WorldMapManager. Manual calls can lead to inconsistent state, race conditions, or redundant network traffic.

## Data Pipeline
The PrefabMarkerProvider acts as a transformation step in the flow of data from game state to the client's user interface.

> Flow:
> PrefabEditSession (State Change) -> WorldMapManager (System Tick) -> **PrefabMarkerProvider.update()** -> MapMarker (Data Object) -> WorldMapTracker -> Network Packet -> Client World Map (UI Update)

