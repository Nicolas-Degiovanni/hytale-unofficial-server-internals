---
description: Architectural reference for LocationRadiusProvider
---

# LocationRadiusProvider

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.worldlocationproviders
**Type:** Transient Data Object

## Definition
```java
// Signature
public class LocationRadiusProvider extends WorldLocationProvider {
```

## Architecture & Concepts
The LocationRadiusProvider is a concrete implementation of the WorldLocationProvider strategy. Its primary function is to algorithmically select a new world coordinate on the surface that lies within a specified annulus (a ring-like region) around an initial position.

This class is fundamentally a data-driven configuration object. It is not intended for manual instantiation but is instead deserialized from adventure mode data files by the engine's serialization framework. The static CODEC field defines this contract, specifying the expected data keys (MinRadius, MaxRadius), data types, and validation rules. This declarative approach ensures that all instances are valid upon creation, enforcing constraints such as MinRadius being less than or equal to MaxRadius.

The core logic of its operation involves two distinct steps:
1.  **2D Placement:** A random point is chosen within the configured annulus using polar coordinates (a random angle and a random radius). This determines the X and Z coordinates.
2.  **3D Projection:** The system then queries the live game World to find the highest solid block (the surface height) at the newly calculated XZ coordinate. This projects the 2D point onto the 3D world geometry, producing the final Vector3i result.

This design decouples the *intent* (find a point in a radius) from the *implementation*, allowing game designers to specify location-finding behavior in data files without writing code.

## Lifecycle & Ownership
-   **Creation:** An instance is created exclusively by the Hytale Codec system when parsing a parent configuration file, such as a quest or objective definition. The public constructor exists only to satisfy the codec's reflection requirements.
-   **Scope:** The object's lifetime is bound to the parent configuration object that defines it. It is a transient, stateless object used for a single type of calculation.
-   **Destruction:** The object is managed by the Java garbage collector. It is eligible for collection as soon as its parent configuration object is unloaded or goes out of scope. No manual cleanup is required.

## Internal State & Concurrency
-   **State:** The internal state consists of minRadius and maxRadius. This state is considered immutable after the object is deserialized and validated by the CODEC. The object itself does not cache any data or change its state during its lifetime.

-   **Thread Safety:** This class is conditionally thread-safe. The object's internal fields are read-only after construction, making the instance itself safe to share. However, the primary method, runCondition, accepts a World object. All interactions with the World, specifically calls to getChunk and getHeight, **must** be performed on the main server thread or a thread that has guaranteed safe, synchronous access to world data. Calling runCondition from an asynchronous worker thread without proper synchronization will lead to severe concurrency violations and server instability.

## API Surface
The public API is minimal, exposing only the core functionality required by the adventure system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| runCondition(World world, Vector3i position) | Vector3i | O(1) | Calculates a new surface-level position. Complexity of the underlying world height lookup is not included but is generally fast. Throws exceptions if world data is not loaded for the target coordinates. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly by most developers. It is configured in a data file and used by a higher-level system, such as a quest objective, which resolves the WorldLocationProvider and calls runCondition.

A conceptual example of how the engine might use a configured provider:

```java
// Engine-level code (conceptual)
// 'provider' is an instance of WorldLocationProvider, deserialized from a config file.
// In this case, it would be an instance of LocationRadiusProvider.

WorldLocationProvider provider = objective.getLocationProvider();
Vector3i startPosition = player.getPosition();
World world = player.getWorld();

// Find a new location based on the provider's logic
Vector3i targetLocation = provider.runCondition(world, startPosition);

// The engine now uses targetLocation for the quest
spawnEntityAt(targetLocation);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new LocationRadiusProvider()`. This bypasses the critical validation logic defined in the CODEC and results in an object with default, and likely incorrect, radius values. Always define providers in data files.
-   **Off-Thread World Access:** Never call `runCondition` from an asynchronous task or a different thread without synchronizing with the main world tick. The `World` object is not thread-safe, and this will cause chunk loading race conditions or ConcurrentModificationExceptions.
-   **State Mutation:** Although the fields are not final, do not attempt to modify `minRadius` or `maxRadius` via reflection after instantiation. The object is designed to be immutable.

## Data Pipeline
The flow of data from configuration to a usable world coordinate is a clear, one-way process.

> Flow:
> Adventure Script (JSON/HOCON) -> Engine Codec Deserializer -> **LocationRadiusProvider Instance** -> Quest System invokes `runCondition` -> Final `Vector3i` World Coordinate

