---
description: Architectural reference for WorldSpawnWrapper
---

# WorldSpawnWrapper

**Package:** com.hypixel.hytale.server.spawning.world.manager
**Type:** Transient

## Definition
```java
// Signature
public class WorldSpawnWrapper extends SpawnWrapper<WorldNPCSpawn> {
```

## Architecture & Concepts
The WorldSpawnWrapper class is a behavioral wrapper around a static data asset, specifically a WorldNPCSpawn configuration. It serves as a critical bridge between raw configuration data loaded from game assets and the live, dynamic server spawning system.

Its primary architectural role is to provide a consistent, object-oriented interface for spawn evaluation logic. Instead of passing raw data structures throughout the spawning pipeline, the system encapsulates each spawn definition within a wrapper. This specific implementation, WorldSpawnWrapper, adds behaviors relevant to entities spawning in the main world, such as calculating modifiers based on environmental factors like the moon phase.

By extending the generic SpawnWrapper, it participates in a polymorphic system where different types of spawns (e.g., world, dungeon, quest) can be treated uniformly by the core spawning engine, while still allowing for specialized evaluation logic.

### Lifecycle & Ownership
- **Creation:** Instances are not created directly by gameplay logic. They are instantiated by a higher-level manager, likely a WorldSpawnManager, during the server's asset loading phase. The manager reads all WorldNPCSpawn definitions and creates a corresponding WorldSpawnWrapper for each, populating a central registry.
- **Scope:** An instance of WorldSpawnWrapper persists for the entire server session. It is part of the server's static, in-memory representation of game content.
- **Destruction:** Instances are destroyed and garbage collected only when the server shuts down or performs a full content reload, at which point the managing registry is cleared.

## Internal State & Concurrency
- **State:** The class is effectively **Immutable**. Its internal state is composed of a reference to the WorldNPCSpawn asset and an integer index, both of which are set in the constructor and never modified. All method calls are pure functions that compute results based on this immutable state and the arguments provided.
- **Thread Safety:** This class is inherently **Thread-Safe**. Due to its immutable nature, instances can be safely shared and accessed by multiple threads without any need for external locking or synchronization. This is crucial in a multi-threaded server environment where different world regions may be ticked concurrently.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WorldSpawnWrapper(spawn) | Constructor | O(1) | Constructs the wrapper for a given spawn asset. Retrieves and stores the asset's unique integer index for fast lookups. |
| getMoonPhaseWeightModifier(moonPhase) | double | O(1) | Calculates the spawn weight multiplier for a given moon phase. Returns 1.0 if no modifiers are defined, or 0.0 if the phase is out of bounds. |

## Integration Patterns

### Standard Usage
A developer should never instantiate or interact with this class directly. It is an internal component of the spawning system. The system's spawn evaluation engine retrieves collections of these wrappers from a manager and invokes their methods to determine which NPC to spawn.

```java
// Hypothetical usage within a Spawning Service
WorldSpawnManager manager = context.getService(WorldSpawnManager.class);
List<WorldSpawnWrapper> candidates = manager.getSpawnsForBiome(currentBiome);
int currentMoonPhase = world.getMoonPhase();

for (WorldSpawnWrapper candidate : candidates) {
    // The engine uses the wrapper's API to evaluate spawn conditions
    double weight = candidate.getBaseWeight(); // Inherited from SpawnWrapper
    weight *= candidate.getMoonPhaseWeightModifier(currentMoonPhase);

    if (shouldSpawn(weight)) {
        // ... trigger entity spawn
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new WorldSpawnWrapper()`. These objects are part of a managed collection created at server startup. Ad-hoc instantiation bypasses the central registry and will not be recognized by the spawning engine.
- **State Mutation:** Do not attempt to retrieve the underlying WorldNPCSpawn object and modify its state at runtime. The entire spawning system relies on the assumption that these configurations are static and read-only after being loaded. Runtime modification will lead to race conditions and unpredictable behavior.

## Data Pipeline
WorldSpawnWrapper acts as a processing and enrichment node within the broader server spawning pipeline. It does not originate data but rather transforms and validates it based on dynamic game state.

> Flow:
> Server Tick -> Spawn System Evaluation -> Query for spawn candidates in a given area -> **WorldSpawnWrapper** is evaluated -> `getMoonPhaseWeightModifier` is called with current world state -> A final spawn weight is returned -> Spawn System makes a weighted random decision -> Entity is created in the world

