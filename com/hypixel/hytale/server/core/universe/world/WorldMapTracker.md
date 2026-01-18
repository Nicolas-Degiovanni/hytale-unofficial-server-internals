---
description: Architectural reference for WorldMapTracker
---

# WorldMapTracker

**Package:** com.hypixel.hytale.server.core.universe.world
**Type:** Transient

## Definition
```java
// Signature
public class WorldMapTracker implements Tickable {
```

## Architecture & Concepts

The WorldMapTracker is a session-scoped component responsible for managing the state of the world map for a single, specific player. It acts as the critical server-side counterpart to the client's map user interface. Its primary function is to serve as a stateful filter and data provider, mediating between the global WorldMapManager—which holds the canonical data for all map imagery—and the individual player, who only requires a small, relevant subset of this data at any given time.

As an implementation of the Tickable interface, an instance of WorldMapTracker is an active participant in the server's main game loop. On each tick, it performs two key operations:

1.  **Map Tile Management:** It tracks the player's position and determines which map image chunks should be visible. It employs a spiral-loading algorithm to load new chunks around the player, providing a responsive user experience. It is also responsible for unloading chunks that are no longer in the player's view radius to conserve client memory and server bandwidth.
2.  **Point of Interest (Marker) Management:** It orchestrates the display of dynamic markers, such as other players, quest objectives, or custom points of interest. It queries various MarkerProvider systems and calculates a delta between the markers currently visible to the client and the markers that *should* be visible, sending efficient update packets containing only the changes.

This class is fundamental to the world map system, ensuring that each player receives a personalized, efficient, and up-to-date view of the world around them.

### Lifecycle & Ownership
- **Creation:** A WorldMapTracker is instantiated and bound to a Player entity when that player successfully joins a World. It is not a standalone service but a component whose lifecycle is directly tied to a player's session.
- **Scope:** The instance persists for the duration of the player's presence in a single World. If a player changes dimensions or worlds, its current tracker is destroyed and a new one is created for the new world context.
- **Destruction:** The object is eligible for garbage collection when the associated Player entity is removed from the World, typically upon disconnection or world transfer. The clear method is invoked to send a final packet to the client, ensuring the map is wiped clean.

## Internal State & Concurrency
- **State:** This class is highly stateful and mutable. Its core state includes:
    - **loaded:** A set of long-encoded chunk coordinates that the server believes have been successfully sent to and loaded by the client.
    - **sentMarkers:** A map of marker IDs to MapMarker objects, representing the current set of points of interest visible on the client's map.
    - **pendingReloadChunks:** A set of chunks that have been invalidated and are queued for regeneration and resending.
    - Various timers and position trackers to throttle network updates and optimize data transmission.

- **Thread Safety:** This class is **not thread-safe** for external callers. All public methods, especially tick, must be invoked from the main server thread. However, it contains internal concurrency controls to manage asynchronous operations.
    - A ReentrantReadWriteLock, `loadedLock`, protects the `loaded` and `pendingReloadChunks` sets. This is critical because map image generation can occur on background threads via WorldMapManager, and the lock prevents race conditions between the main thread processing the tick and worker threads completing image futures.
    - The `sentMarkers` map is a ConcurrentHashMap, allowing different MarkerProvider systems to potentially operate concurrently, although the final diffing and packet creation in `updatePointsOfInterest` occurs synchronously within the tick.

**WARNING:** Do not access or modify the internal collections from any thread other than the one executing the tick method without a deep understanding of the locking mechanism. Direct manipulation will lead to state corruption and network desynchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(float dt) | void | O(N) | The main entry point from the game loop. Orchestrates all map and marker updates. Complexity is proportional to the number of chunks at the edge of the view radius. |
| updateCurrentZoneAndBiome(...) | void | O(1) | Callback to update the player's current location context, potentially triggering zone discovery events. |
| trySendMarker(...) | void | O(1) | Interface for MarkerProvider systems to submit a marker for potential display. The tracker determines visibility and state changes. |
| clear() | void | O(L) | Resets all internal state and sends a ClearWorldMap packet to the client. Complexity is proportional to the number of loaded chunks (L). |
| clearChunks(LongSet) | void | O(C) | Invalidates a specific set of map chunks, queueing them for a future reload. C is the size of the input set. |
| sendSettings(World) | void | O(1) | Asynchronously sends the current world map settings (e.g., teleport permissions) to the client. |
| discoverZone(World, String) | boolean | O(1) | Persists a zone as "discovered" for the player and updates client settings. |
| setViewRadiusOverride(Integer) | void | O(1) | Overrides the default map view radius for this player, forcing a full map clear and reload. |

## Integration Patterns

### Standard Usage

The WorldMapTracker is not intended to be directly retrieved or manipulated by general gameplay code. Its primary interaction is with the core server loop and specialized `MarkerProvider` systems. A `MarkerProvider` updates markers during the `updatePointsOfInterest` phase of the tick.

```java
// Hypothetical MarkerProvider implementation
public class PlayerMarkerProvider implements WorldMapManager.MarkerProvider {
    @Override
    public void update(World world, GameplayConfig config, WorldMapTracker tracker, int viewRadius, int playerChunkX, int playerChunkZ) {
        // Get the player this tracker belongs to
        Player trackingPlayer = tracker.getPlayer();

        // Find other nearby players
        for (Player otherPlayer : world.getPlayersInRadius(trackingPlayer.getPosition())) {
            if (trackingPlayer.canSee(otherPlayer)) {
                // Use the tracker's API to submit the marker
                tracker.trySendMarker(
                    viewRadius,
                    playerChunkX,
                    playerChunkZ,
                    otherPlayer.getPosition(),
                    otherPlayer.getYaw(),
                    otherPlayer.getUuid().toString(),
                    otherPlayer.getDisplayName(),
                    otherPlayer,
                    (id, name, p) -> createPlayerMarker(id, name, p) // Lambda to create the marker object
                );
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new WorldMapTracker()`. The tracker is tightly coupled to a player's lifecycle and is managed by the server's player management systems. Manual creation will result in a disconnected, non-functional object.
- **Cross-Thread Modification:** Do not call methods like `tick`, `clear`, or `clearChunks` from asynchronous tasks or different threads. These methods are not thread-safe and expect to be run on the main server thread.
- **Holding References:** Avoid storing long-lived references to a WorldMapTracker. Since its lifecycle is tied to a player's session in a specific world, holding a reference beyond that scope can lead to memory leaks.

## Data Pipeline

The flow of data is orchestrated by the `tick` method, transforming world state and player actions into compressed network packets for the client.

> Flow:
> 1.  **Inputs:**
>     - Player Position (from TransformComponent)
>     - WorldMapManager (Source of truth for map image data)
>     - Various MarkerProvider systems (Source of points of interest)
> 2.  **Processing (WorldMapTracker):**
>     - Calculates visible chunk area based on player position and view radius.
>     - Diffs visible chunks against the `loaded` set.
>     - Requests new map images asynchronously from WorldMapManager.
>     - Collects results from MarkerProviders.
>     - Diffs new markers against the `sentMarkers` map.
>     - Batches chunk and marker changes into packets, respecting network size limits.
> 3.  **Outputs:**
>     - UpdateWorldMap packets (containing new map tiles and marker deltas)
>     - ClearWorldMap packets (on world change or full reset)
>     - UpdateWorldMapSettings packets (on permission or setting changes)
> 4.  **Destination:**
>     - PlayerConnection (Network Layer) -> Client Game Instance

