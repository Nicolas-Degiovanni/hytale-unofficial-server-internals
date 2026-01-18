---
description: Architectural reference for PortalSpawn
---

# PortalSpawn

**Package:** com.hypixel.hytale.server.core.asset.type.portalworld
**Type:** Configuration Model

## Definition
```java
// Signature
public class PortalSpawn {
```

## Architecture & Concepts
The PortalSpawn class is a pure data container that defines the parameters for finding a valid player spawn location within a world. It is not a service or a manager; it is a schema for a configuration block that dictates the behavior of the server's spawn-finding algorithm.

Its primary architectural feature is the static **CODEC** field. This links the class directly to the engine's serialization system, allowing instances to be loaded from asset files (e.g., JSON definitions for a specific world or dimension). The class encapsulates the rules for a probabilistic search, using parameters like radius, scan height, and attempt counts to locate a suitable block for a player to spawn on.

This object is consumed by world generation or player-spawning services, which interpret its properties to execute the search. It effectively decouples the spawn-finding *logic* from the spawn-finding *configuration*.

## Lifecycle & Ownership
- **Creation:** A PortalSpawn instance is created exclusively by the engine's **Codec** deserialization pipeline when a parent asset (such as a portal world definition) is loaded from disk. The public constructor exists only to satisfy the requirements of the BuilderCodec.

- **Scope:** The object's lifetime is bound to the asset that defines it. It persists in memory as long as its parent world definition is loaded and referenced by the server.

- **Destruction:** The object is a simple data holder with no external resources. It is eligible for garbage collection once the parent asset is unloaded and all references to the instance are released. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** The state of a PortalSpawn instance is **effectively immutable**. While its fields are not marked as final, there are no public setters. The state is populated once during deserialization via the CODEC and is not intended to be modified thereafter. All fields are initialized with sensible defaults.

- **Thread Safety:** This class is **thread-safe for reads**. As its state is established at creation and does not change, multiple threads (e.g., different world generation threads) can safely access its properties via the getter methods without requiring any synchronization or locks.

## API Surface
The public contract is composed of a static codec for instantiation and a set of getters for reading configuration values.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| **CODEC** | BuilderCodec | N/A | **Critical.** The static codec used by the engine to serialize and deserialize PortalSpawn instances from asset files. |
| getCenter() | Vector3i | O(1) | Retrieves the central coordinate around which the spawn search is conducted. |
| getCheckSpawnY() | int | O(1) | Returns the starting Y-level for the downward scan to find a valid surface. |
| getScanHeight() | int | O(1) | Returns the maximum number of blocks to scan downwards from the CheckSpawnY. |
| getMinRadius() | int | O(1) | Returns the minimum distance from the center for a spawn candidate. |
| getMaxRadius() | int | O(1) | Returns the maximum distance from the center for a spawn candidate. |
| getChunkDartThrows() | int | O(1) | Returns the number of attempts the algorithm makes to select a random chunk. |
| getChecksPerChunk() | int | O(1) | Returns the number of random location checks performed within a selected chunk. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly. Instead, its values are defined in an external asset file and consumed by a higher-level service. A developer's primary interaction is with the asset file, not the Java class.

The following conceptual example shows how a service might consume a pre-loaded PortalSpawn instance.

```java
// A hypothetical service that uses the PortalSpawn configuration
public class SpawnPointFinder {
    public Vector3i findSpawnLocation(World world, PortalSpawn spawnConfig) {
        // Logic uses the config to find a point
        Vector3i center = spawnConfig.getCenter();
        int minRadius = spawnConfig.getMinRadius();
        int maxRadius = spawnConfig.getMaxRadius();
        
        // ... perform dart throws and surface checks based on config ...
        
        return foundSpawnPoint;
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new PortalSpawn()`. This bypasses the asset loading system and results in an object with default values, which will almost certainly lead to incorrect or unexpected spawn behavior. The object must be loaded via the asset pipeline.

- **State Mutation:** Do not use reflection to modify the fields of a PortalSpawn instance after it has been created. The system relies on this object being an immutable data source during its lifetime. Modifying it at runtime can lead to non-deterministic behavior and race conditions.

## Data Pipeline
The PortalSpawn class is a passive component in the server's data flow, acting as a structured container for configuration that is loaded from disk.

> Flow:
> World Asset File (e.g., world.json) -> Engine AssetLoader -> **Codec Deserializer** -> Populates **PortalSpawn Instance** -> SpawnPointFinder Service -> Calculates Spawn Location

