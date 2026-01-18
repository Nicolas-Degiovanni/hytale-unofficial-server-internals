---
description: Architectural reference for WorldMapSettings
---

# WorldMapSettings

**Package:** com.hypixel.hytale.server.core.universe.world.worldmap
**Type:** Configuration Object / Transient

## Definition
```java
// Signature
public class WorldMapSettings {
```

## Architecture & Concepts
The WorldMapSettings class is a server-side configuration object that encapsulates all parameters governing the behavior of the world map feature for a specific world instance. It is not a service or manager, but rather a data-centric container that centralizes rules and boundaries.

Its primary architectural role is to act as a bridge between static world configuration (likely loaded from files) and the network protocol. The class holds a pre-constructed network packet, UpdateWorldMapSettings, which is sent to clients to synchronize their world map capabilities with the server's rules. This design decouples the world loading and configuration logic from the network and player session management logic.

The static constant, DISABLED, implements a Null Object Pattern, providing a safe, non-null, and inert default for worlds where the map feature is turned off. This simplifies downstream logic by removing the need for null checks.

## Lifecycle & Ownership
-   **Creation:** An instance of WorldMapSettings is typically created by a higher-level authority, such as a World or Universe manager, during the world loading and initialization sequence. Its constructor is populated with values sourced from the world's configuration files. The special DISABLED instance is created once at class-loading time.
-   **Scope:** The object's lifetime is tightly coupled to the world it configures. It persists in memory for the entire duration that the world is active and loaded on the server.
-   **Destruction:** The object contains no unmanaged resources and has no explicit destruction method. It becomes eligible for garbage collection when the corresponding world is unloaded and all references to the settings object are released.

## Internal State & Concurrency
-   **State:** The object is **Effectively Immutable**. While its fields are not declared final, there are no public mutator methods (setters). State is established exclusively through the constructor at the time of creation. This write-once, read-many pattern ensures that the world map configuration remains consistent throughout the world's lifecycle.

-   **Thread Safety:** The class is **Conditionally Thread-Safe**. Because its state is fixed after construction, it can be safely read by multiple threads simultaneously (e.g., multiple player threads checking map boundaries). However, direct modification of the underlying UpdateWorldMapSettings packet object returned by getSettingsPacket is not thread-safe and is a critical anti-pattern. The getViewRadius method is a pure function with no side effects and is inherently safe for concurrent access.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWorldMapArea() | Box2D | O(1) | Returns the 2D bounding box that defines the valid, viewable area of the world map. |
| getImageScale() | float | O(1) | Retrieves the server-defined scale factor for the map imagery. |
| getSettingsPacket() | UpdateWorldMapSettings | O(1) | Returns the fully-formed network packet used to transmit these settings to a client. |
| getViewRadius(int viewRadius) | int | O(1) | Calculates the effective view radius for a player, applying the server's multiplier and clamping the result within the configured min/max bounds. |

## Integration Patterns

### Standard Usage
WorldMapSettings should be instantiated once during world startup and stored by a central world management service. Other systems then retrieve this canonical instance to perform their work.

```java
// During world initialization
Box2D mapBounds = loadMapBoundsFromConfig();
WorldMapSettings settings = new WorldMapSettings(mapBounds, 0.5f, 1.0f, ...);
currentWorld.setMapSettings(settings);

// When a player joins the world
PlayerConnection player = ...;
WorldMapSettings settings = currentWorld.getMapSettings();
player.sendPacket(settings.getSettingsPacket());
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation:** Do not retrieve the settings packet and modify its contents. This breaks the effectively immutable contract of the parent object and can lead to state desynchronization for players who join later.
    ```java
    // BAD: Mutating shared state
    WorldMapSettings settings = world.getMapSettings();
    settings.getSettingsPacket().enabled = false; // This affects all subsequent calls
    ```
-   **Redundant Instantiation:** Avoid creating new instances of WorldMapSettings within game loops or per-player logic. It is a configuration object that represents a fixed state for the entire world and should be treated as a singleton for that world's scope.

## Data Pipeline
This class primarily serves as a data source at the beginning of a configuration pipeline that flows from the server's configuration to the game client.

> Flow:
> World Configuration File -> World Loader -> **WorldMapSettings** (Instantiation) -> Player Session Manager -> Network Dispatcher -> Client Rendering Engine

