---
description: Architectural reference for PortalSpawnFinder
---

# PortalSpawnFinder

**Package:** com.hypixel.hytale.builtin.portals.ui
**Type:** Utility

## Definition
```java
// Signature
public final class PortalSpawnFinder {
```

## Architecture & Concepts
The PortalSpawnFinder is a stateless utility class responsible for determining a safe and valid spawn location for an entity entering a new world via a portal. It acts as a low-level spatial query engine that operates directly on world chunk data to satisfy placement constraints defined by a PortalSpawn configuration asset.

Its core design is a two-tiered search strategy to balance performance and reliability:

1.  **Probabilistic "Dart-Throw" Search:** The primary strategy, `findSpawnByThrowingDarts`, attempts to find an ideal spawn point. It randomly samples horizontal coordinates within a configured annulus (a ring defined by a minimum and maximum radius). For each sampled point, it performs a vertical scan downwards from a specified altitude to find the first solid ground with sufficient empty space above it. This method is designed to find varied and aesthetically pleasing spawn locations in open areas.

2.  **Deterministic Fallback Search:** If the primary strategy fails after a set number of attempts, `findFallbackPositionOnGround` is invoked as a failsafe. This method performs a simple, exhaustive vertical scan from the top of the world downwards at a fixed horizontal coordinate (the center of the world or a configured point). Its purpose is to guarantee a spawn location, even in difficult or constrained terrain, preventing the entity from spawning in the void.

This class directly interfaces with the world's chunk storage system, reading block and fluid data at a granular level to make its decisions.

## Lifecycle & Ownership
- **Creation:** As a static utility class, PortalSpawnFinder is never instantiated. Its methods are invoked directly on the class.
- **Scope:** The operational scope is transient, limited to the duration of a single static method call, typically during a world-loading or player-teleportation event.
- **Destruction:** Not applicable. The class is part of the application's static memory and is unloaded only when the JVM shuts down.

## Internal State & Concurrency
- **State:** PortalSpawnFinder is entirely stateless. All required data, such as the target World and PortalSpawn configuration, are passed as arguments to its methods. It maintains no internal fields or caches.
- **Thread Safety:** The class is thread-safe. It performs read-only operations on the provided World object and uses ThreadLocalRandom for its probabilistic searches to avoid contention. It is safe to call `computeSpawnTransform` from multiple threads simultaneously, provided the underlying World data structures support concurrent reads.

**WARNING:** While the class itself is thread-safe, executing a search with a large radius or a high number of checks can be computationally expensive. Invoking this on a performance-critical thread, like the main server tick loop, may introduce stutter. It is recommended to execute this utility within an asynchronous task or a dedicated world-generation thread.

## API Surface
The public contract consists of a single static method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeSpawnTransform(World, PortalSpawn) | Transform | Bounded Search | The primary entry point. Orchestrates the dart-throw and fallback search strategies to find a valid spawn location. Returns a Transform object with position and orientation, or null if no valid location can be found. |

## Integration Patterns

### Standard Usage
PortalSpawnFinder should be invoked by higher-level systems managing world transitions or entity placement. The caller provides the target world and the rules for the search.

```java
// A service responsible for player teleportation
// obtains the target world and its spawn configuration.
World targetWorld = ...;
PortalSpawn spawnConfig = ...;

// Compute the spawn location. This may be a blocking call.
Transform spawnPoint = PortalSpawnFinder.computeSpawnTransform(targetWorld, spawnConfig);

if (spawnPoint != null) {
    player.teleportTo(spawnPoint);
} else {
    // Handle the critical failure case where no spawn could be found.
    HytaleLogger.getLogger().at(Level.SEVERE).log("Failed to find any spawn point in world!");
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Null Return:** The `computeSpawnTransform` method can return null. Failure to handle this case will result in a NullPointerException and can leave an entity in an invalid state (e.g., disconnected or trapped in the void).
- **Synchronous Execution on Hot Paths:** Do not call this method on the main server thread if the PortalSpawn configuration involves a large search radius or a high number of checks. This can block the thread and cause server-wide lag. Offload the call to a worker thread.
- **Poor Configuration:** Relying on the fallback mechanism indicates that the primary dart-throw search parameters in the PortalSpawn asset are too restrictive for the target world's terrain. This should be treated as a configuration error to be fixed, not a normal operational path.

## Data Pipeline
The flow of data and logic through the component is a conditional, multi-stage search.

> **Flow:**
> 1.  **Input:** A `World` object and a `PortalSpawn` configuration are passed to `computeSpawnTransform`.
> 2.  **Primary Search (`findSpawnByThrowingDarts`):**
>     - A horizontal position is randomly sampled within the configured radius.
>     - A vertical scan (`findGroundWithinChunk`) is initiated from `checkSpawnY` downwards.
>     - The `getMaterial` helper queries raw chunk data to identify SOLID, AIR, or FLUID blocks.
>     - If a candidate ground position is found, it is validated by `FitsAPortal` to ensure enough clearance.
>     - If successful, a `Vector3d` is returned.
> 3.  **Fallback Search (`findFallbackPositionOnGround`):**
>     - If the Primary Search fails after all attempts, this is triggered.
>     - A non-random, exhaustive vertical scan is performed from the world ceiling at a fixed point.
>     - It accepts fluids as a valid standing surface, increasing the chance of success.
>     - If successful, a `Vector3d` is returned.
> 4.  **Transform Creation:**
>     - The final `Vector3d` position is converted into a `Transform`, which includes a calculated orientation (facing towards the spawn point's center).
> 5.  **Output:** The final `Transform` is returned. If both searches fail, `null` is returned.

