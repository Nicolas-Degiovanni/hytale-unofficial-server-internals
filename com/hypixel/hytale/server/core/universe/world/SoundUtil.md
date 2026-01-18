---
description: Architectural reference for SoundUtil
---

# SoundUtil

**Package:** com.hypixel.hytale.server.core.universe.world
**Type:** Utility

## Definition
```java
// Signature
public class SoundUtil {
```

## Architecture & Concepts

The SoundUtil class is a stateless, server-side utility that provides a high-level API for triggering audio events for clients. It serves as the primary bridge between server-side game logic and the client-side audio engine, abstracting the complexities of network packet creation and player proximity filtering.

Its core responsibilities include:

1.  **Packet Construction:** Translating high-level requests (e.g., "play explosion sound at X,Y,Z") into specific network packets such as PlaySoundEvent2D, PlaySoundEvent3D, and PlaySoundEventEntity.
2.  **Client Targeting:** Determining which players should receive an audio event. This ranges from sending a packet to a single player, broadcasting to all players, or performing sophisticated spatial queries to target only players within audible range of a 3D sound.
3.  **ECS Integration:** It operates deeply within the server's Entity-Component-System (ECS) framework. All operations require a ComponentAccessor to resolve entity data, player connections (PlayerRef), and entity positions (TransformComponent).
4.  **Performance Optimization:** For 3D sounds, SoundUtil leverages the server's spatial partitioning system (SpatialResource) to execute efficient proximity queries. This avoids iterating over every player in the world, which is critical for performance in a massively multiplayer environment.

This class is not an audio engine itself; it is a command dispatcher. The server does not process or play any audio. Its sole function is to instruct clients *what* sound to play and *where*.

## Lifecycle & Ownership

-   **Creation:** As a utility class composed exclusively of static methods, SoundUtil is never instantiated. The class is loaded into the JVM by the class loader upon its first use.
-   **Scope:** The class and its static methods are globally accessible throughout the server's runtime.
-   **Destruction:** The class is unloaded from memory only when the Hytale server process terminates and the JVM shuts down. There is no instance-level state to manage or clean up.

## Internal State & Concurrency

-   **State:** SoundUtil is **stateless**. It contains no static or instance fields to store data between method invocations. All required context, such as the world state via ComponentAccessor, is passed in as method arguments. The class is therefore inherently immutable.

-   **Thread Safety:** The methods are conditionally thread-safe. While the class itself has no mutable state, its operations are dependent on the thread safety of the provided ComponentAccessor. The use of `SpatialResource.getThreadLocalReferenceList()` is a strong indicator that the underlying spatial query system is optimized for thread-local operations.

    **WARNING:** It is assumed that a given ComponentAccessor instance is accessed from a single thread, typically the main world-tick thread. Passing the same ComponentAccessor to SoundUtil from multiple concurrent threads will lead to undefined behavior and likely cause severe concurrency violations within the ECS framework. All calls to SoundUtil must be synchronized by the calling system if multi-threaded access to the same game state is required.

## API Surface

The API is designed around three distinct types of sound events: 2D (UI/ambient), 3D (world-positioned), and Entity-attached.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| playSoundEventEntity(int, int, ComponentAccessor) | void | O(P) | Broadcasts a request for clients to play a sound attached to a specific entity network ID. P is the total number of players. |
| playSoundEvent2dToPlayer(PlayerRef, int, SoundCategory) | void | O(1) | Sends a non-spatial sound packet to a single, specific player. Used for UI feedback or private audio cues. |
| playSoundEvent3d(int, SoundCategory, Vector3d, ComponentAccessor) | void | O(log N + K) | Performs a spatial query to find all players K within the max distance of a sound event and sends them a packet to play a sound at a world coordinate. N is the total number of players in the spatial index. |
| playSoundEvent3d(..., Predicate, ComponentAccessor) | void | O(log N + K) | An advanced 3D sound variant that applies an additional Predicate filter to the players found by the spatial query, allowing for complex line-of-sight or team-based audio logic. |

## Integration Patterns

### Standard Usage

SoundUtil must be called from a system that has access to a valid ComponentAccessor for the current world state, typically within an ECS system's update loop.

```java
// Example: A system that creates an explosion effect
void createExplosion(ComponentAccessor<EntityStore> accessor, Vector3d position) {
    // Get the sound event index from an asset manager or config
    int explosionSoundIndex = SoundEvents.GENERIC_EXPLOSION;

    // Play the sound at the explosion's location for all nearby players
    SoundUtil.playSoundEvent3d(
        explosionSoundIndex,
        SoundCategory.SFX,
        position,
        accessor
    );
}
```

### Anti-Patterns (Do NOT do this)

-   **Asynchronous Execution:** Do not call SoundUtil methods from a separate, long-running thread without proper synchronization or a ComponentAccessor designed for that context. The data read from the accessor may become stale or cause race conditions with the main game thread.
-   **Ignoring Sound Index Zero:** The methods contain a guard clause `if (soundEventIndex != 0)`. Zero is treated as a sentinel for "no sound". Game logic should avoid calling these methods with an index of zero, as it is a wasteful no-op.
-   **Incorrect Sound Type:** Do not use `playSoundEvent2d` for a sound that clearly originates from a point in the world (e.g., a footstep). This breaks immersion, as the sound will not be spatialized for the client. Conversely, do not use `playSoundEvent3d` for UI sounds.

## Data Pipeline

The flow of data for the most common use case, a 3D positional sound, illustrates the class's role in the engine. It transforms a server-side game event into a series of targeted network packets.

> Flow:
> Game Logic Trigger (e.g., Block Destruction) -> **SoundUtil.playSoundEvent3d** -> Asset System (fetches SoundEvent max distance) -> SpatialResource Query (finds nearby players) -> Packet Construction (new PlaySoundEvent3D) -> PlayerRef Component (retrieves PacketHandler) -> Network Layer (packet sent to each targeted client)

