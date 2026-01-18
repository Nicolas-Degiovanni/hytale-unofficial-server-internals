---
description: Architectural reference for FitToHeightMapSpawnProvider
---

# FitToHeightMapSpawnProvider

**Package:** com.hypixel.hytale.server.core.universe.world.spawn
**Type:** Transient

## Definition
```java
// Signature
public class FitToHeightMapSpawnProvider implements ISpawnProvider {
```

## Architecture & Concepts
The FitToHeightMapSpawnProvider is an implementation of the **Decorator Pattern**. It is not a standalone spawn provider but rather a wrapper that modifies the behavior of another, underlying ISpawnProvider implementation.

Its primary function is to act as a post-processor for spawn point generation. It takes a spawn point calculated by its wrapped provider and ensures the vertical position is valid relative to the world's terrain. Specifically, if a proposed spawn point is below the world (Y < 0), this class corrects it by "snapping" the Y-coordinate to the surface height at that location, preventing players from spawning underground or falling through the world.

This component is a critical safeguard in the player spawning pipeline, providing robustness for spawn providers that are not terrain-aware, such as those that use fixed, hard-coded coordinates.

## Lifecycle & Ownership
-   **Creation:** Instantiation occurs in two primary ways:
    1.  **Declaratively:** Deserialized from world configuration files via its static CODEC field. The server's world-loading system constructs it and its nested spawn provider based on the configuration.
    2.  **Programmatically:** Manually instantiated by the server logic, wrapping another ISpawnProvider instance. For example: `new FitToHeightMapSpawnProvider(new FixedSpawnProvider(...))`.

-   **Scope:** Its lifetime is bound to the component that holds it, typically the World instance it is configured for. It is not a global singleton and persists as long as its parent World object.

-   **Destruction:** The object is eligible for garbage collection when the World it belongs to is unloaded. It requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** The class holds a single reference to the wrapped ISpawnProvider. This reference is set at construction and is not modified, making the object's configuration effectively immutable. It does not cache any world data or maintain other mutable state.

-   **Thread Safety:** This class is conditionally thread-safe. The core logic in getSpawnPoint reads world data via getNonTickingChunk, which is designed to be a safe way to access chunk data from outside the main world tick thread. However, the overall thread safety is dependent on the wrapped ISpawnProvider.
    -   **Warning:** The getSpawnPoint method directly modifies the Transform object returned by the wrapped provider. Callers should be aware of this side effect, as the original Transform object is mutated in place.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSpawnPoint(World, UUID) | Transform | O(1) | Retrieves a spawn point from the wrapped provider and adjusts its Y-coordinate to the terrain height if it is below Y=0. Complexity assumes chunk and heightmap lookups are constant time. |
| getSpawnPoints() | Transform[] | Delegated | Returns all possible spawn points from the wrapped provider. Complexity is determined by the wrapped implementation. |
| isWithinSpawnDistance(Vector3d, double) | boolean | Delegated | Checks if a position is within the spawn area, as defined by the wrapped provider. Complexity is determined by the wrapped implementation. |

## Integration Patterns

### Standard Usage
This provider is intended to be chained with another provider to add terrain-aware vertical correction. It is most commonly used with providers that define spawn points without knowledge of the generated world geometry.

```java
// Example: Making a fixed spawn point terrain-aware
ISpawnProvider fixedProvider = new FixedSpawnProvider(new Vector3d(100, 64, 100));
ISpawnProvider safeProvider = new FitToHeightMapSpawnProvider(fixedProvider);

// When safeProvider.getSpawnPoint is called, it will correct the Y-level
// of the fixed point to match the terrain at X=100, Z=100.
Transform finalSpawn = safeProvider.getSpawnPoint(world, playerUUID);
```

### Anti-Patterns (Do NOT do this)
-   **Null Delegation:** Do not use the default constructor `new FitToHeightMapSpawnProvider()` without subsequently setting the internal provider. Doing so will result in a NullPointerException when any method is called.
-   **Redundant Wrapping:** Avoid wrapping a provider that is already terrain-aware. This adds unnecessary overhead and can lead to unpredictable behavior if both providers attempt to adjust the spawn height.

## Data Pipeline
The component acts as a filter or a transformation step in the data flow of spawn point generation.

> Flow:
> Spawn Request -> Wrapped ISpawnProvider -> Initial Transform -> **FitToHeightMapSpawnProvider** (Conditional Y-Correction) -> Final Transform -> Player Spawning Logic

