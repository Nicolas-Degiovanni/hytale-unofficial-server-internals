---
description: Architectural reference for ZoneGeneratorResult
---

# ZoneGeneratorResult

**Package:** com.hypixel.hytale.server.worldgen.zone
**Type:** Transient

## Definition
```java
// Signature
public class ZoneGeneratorResult {
```

## Architecture & Concepts
The ZoneGeneratorResult is a fundamental Data Transfer Object (DTO) within the server-side world generation pipeline. Its sole purpose is to encapsulate the output of a spatial query against the world's zone map. It acts as a standardized data contract, decoupling zone calculation logic (the *producer*) from terrain and feature generation systems (the *consumers*).

This class carries two critical pieces of information:
1.  **Zone:** An object representing the specific biome or region at a given coordinate.
2.  **borderDistance:** A double-precision value indicating the distance from the query coordinate to the nearest edge of the identified Zone. This value is essential for procedural generation algorithms that require smooth transitions, such as terrain blending, texture mapping, and density-based feature placement.

By bundling these two values, the ZoneGeneratorResult ensures that consumers have all necessary contextual information from a single, atomic query, preventing redundant calculations.

### Lifecycle & Ownership
-   **Creation:** An instance is created on-demand by a ZoneGenerator or a similar service during the evaluation of a specific world coordinate. It is a direct result of a computation.
-   **Scope:** Method-scoped. The object's lifetime is extremely short, typically confined to the call stack of the method that requested the zone information. It is created, returned, immediately consumed, and then becomes eligible for garbage collection.
-   **Destruction:** Managed exclusively by the Java Garbage Collector. There are no manual cleanup or `close` operations associated with this object.

## Internal State & Concurrency
-   **State:** Highly mutable. The object is a simple container for a Zone reference and a double. Its state is intended to be set once by its creator, either via the constructor or immediately following instantiation with the default constructor. It performs no internal caching or state derivation.

-   **Thread Safety:** **This class is not thread-safe.** It provides no internal synchronization mechanisms. It is designed to be created, populated, and read within the confines of a single thread, such as a dedicated world generation task for a specific chunk.

    **WARNING:** Sharing a ZoneGeneratorResult instance across multiple threads without external, explicit locking will result in severe data corruption and race conditions. Do not store this object in a concurrently accessed field.

## API Surface
The public API consists entirely of simple accessors and mutators for its internal fields.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ZoneGeneratorResult() | constructor | O(1) | Creates an empty result object. |
| ZoneGeneratorResult(Zone, double) | constructor | O(1) | Creates and initializes a result object. |
| getZone() | Zone | O(1) | Retrieves the Zone associated with this result. |
| getBorderDistance() | double | O(1) | Retrieves the calculated distance to the zone's border. |
| setZone(Zone) | void | O(1) | Sets or overwrites the Zone for this result. |
| setBorderDistance(double) | void | O(1) | Sets or overwrites the border distance for this result. |

## Integration Patterns

### Standard Usage
The canonical use case involves a system requesting zone information, receiving a ZoneGeneratorResult, and immediately deconstructing it to inform subsequent generation logic.

```java
// A consumer, such as a terrain generator, queries the ZoneGenerator
ZoneGenerator zoneGen = world.getZoneGenerator();
ZoneGeneratorResult result = zoneGen.getZoneAt(chunkX, chunkZ);

// The result is immediately consumed
Zone currentZone = result.getZone();
double distanceToEdge = result.getBorderDistance();

// Logic then proceeds based on the retrieved values
if (distanceToEdge < 16.0) {
    // Apply terrain blending logic near the zone border...
}
```

### Anti-Patterns (Do NOT do this)
-   **Object Re-use:** Do not attempt to cache and re-use a ZoneGeneratorResult instance for multiple queries. This is a critical error that leads to stale data, as the object is not designed for pooling or reset. A new instance must be returned for every query.
-   **Shared State:** Do not pass a ZoneGeneratorResult instance to another thread or store it in a shared collection. Its mutable design makes it completely unsuitable for concurrent access.
-   **Consumer Instantiation:** Systems that consume zone data should never instantiate this class directly using `new ZoneGeneratorResult()`. It is exclusively a return type produced by a generator service.

## Data Pipeline
ZoneGeneratorResult serves as the data payload that bridges the gap between spatial lookup and procedural content generation.

> Flow:
> World Coordinates (x, z) -> `ZoneGenerator.getZoneAt()` -> **ZoneGeneratorResult** -> Terrain Generation Stage -> Feature Placement Stage

