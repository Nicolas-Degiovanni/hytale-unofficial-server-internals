---
description: Architectural reference for Zone
---

# Zone

**Package:** com.hypixel.hytale.server.worldgen.zone
**Type:** Value Object / Data Transfer Object (DTO)

## Definition
```java
// Signature
public record Zone(
   int id,
   @Nonnull String name,
   @Nonnull ZoneDiscoveryConfig discoveryConfig,
   @Nullable CaveGenerator caveGenerator,
   @Nonnull BiomePatternGenerator biomePatternGenerator,
   @Nonnull UniquePrefabContainer uniquePrefabContainer
) {
```

## Architecture & Concepts
The Zone record is a fundamental, immutable data structure that serves as a blueprint for a large-scale geographical region within the Hytale world. It does not contain any active logic itself; instead, it aggregates the configuration and specialized generator instances required to procedurally generate a cohesive and distinct area of the game world.

Architecturally, Zone acts as a configuration container passed to the core world generation services. It decouples the high-level definition of a world area (e.g., a "Cursed Forest") from the low-level generation algorithms. The world generation pipeline queries a central registry for the appropriate Zone based on world coordinates, and then uses the generators and data held by that Zone instance to build the terrain, biomes, caves, and unique structures for that region.

The nested records, particularly UniqueEntry and UniqueCandidate, are specialized data structures used by the UniquePrefabContainer to manage the placement of large, one-of-a-kind points of interest within the zone's boundaries.

## Lifecycle & Ownership
- **Creation:** Zone instances are not created dynamically during gameplay. They are deserialized from configuration files (e.g., JSON) at server startup by a dedicated loading service, such as a ZoneRegistry or WorldGenConfigLoader. This process populates a central, read-only collection of all available zones.
- **Scope:** A Zone instance is application-scoped. Once loaded, it persists for the entire lifetime of the server process. It is treated as static, canonical data.
- **Destruction:** The object is eligible for garbage collection only upon server shutdown when the application context is torn down.

## Internal State & Concurrency
- **State:** The Zone class is a Java record, making it **deeply immutable** by design. All fields are final and are assigned only at construction time. It holds no mutable state and performs no caching.
- **Thread Safety:** As a fully immutable object, Zone is **inherently thread-safe**. It can be safely read by any number of concurrent world generation threads without locks, mutexes, or any other synchronization primitives. This is a critical design choice that enables highly parallelized and performant world generation.

## API Surface
The primary API consists of the record's component accessors.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| id | int | O(1) | The unique numeric identifier for the zone. |
| name | String | O(1) | The human-readable name of the zone. |
| discoveryConfig | ZoneDiscoveryConfig | O(1) | Configuration related to how players discover this zone. |
| caveGenerator | CaveGenerator | O(1) | The specific cave generation algorithm for this zone. May be null. |
| biomePatternGenerator | BiomePatternGenerator | O(1) | The generator responsible for creating the biome layout within this zone. |
| uniquePrefabContainer | UniquePrefabContainer | O(1) | Manages the placement rules for unique, large-scale prefabs. |

### Nested Type: Zone.UniqueEntry
This record defines the placement rules for a unique prefab.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesParent(color) | boolean | O(N) | Checks if the given color value matches any of the defined parent colors. N is the size of the parent array. |

## Integration Patterns

### Standard Usage
A Zone is never used directly. It is retrieved from a central registry and its components are passed to world generation workers that operate on specific world chunks.

```java
// Pseudo-code for a world generator service
ZoneRegistry zoneRegistry = context.getService(ZoneRegistry.class);
WorldChunk targetChunk = ...;

// 1. Determine the appropriate Zone for a given coordinate
Zone currentZone = zoneRegistry.getZoneForCoordinates(targetChunk.getCoords());

// 2. Use the Zone's aggregated generators to build the world
BiomePattern pattern = currentZone.biomePatternGenerator().generate(seed, targetChunk.getCoords());
if (currentZone.caveGenerator() != null) {
    currentZone.caveGenerator().carveCaves(targetChunk);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never construct a Zone manually using its record constructor. Zones represent canonical game data loaded from configuration. Manual creation will lead to a world state that is inconsistent with the game's design files and will not be recognized by the rest of the engine.
- **Stateful Wrappers:** Do not wrap a Zone instance in a mutable class. The immutability of the Zone is a core architectural guarantee for safe, concurrent world generation. Introducing mutable state around it is a significant risk.

## Data Pipeline
The Zone object is not a processing stage in a pipeline; rather, it is a static configuration input that *defines* the stages of the world generation pipeline for a specific region.

> Flow:
> Game Asset Files (JSON) -> Server Bootstrap -> ZoneRegistry Deserialization -> **Zone Instance** -> WorldGenerator Service -> Final World Chunk Data

