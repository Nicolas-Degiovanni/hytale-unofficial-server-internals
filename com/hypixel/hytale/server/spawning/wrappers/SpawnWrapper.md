---
description: Architectural reference for SpawnWrapper
---

# SpawnWrapper

**Package:** com.hypixel.hytale.server.spawning.wrappers
**Type:** Transient Wrapper

## Definition
```java
// Signature
public abstract class SpawnWrapper<T extends NPCSpawn> {
```

## Architecture & Concepts

The SpawnWrapper is an abstract base class that serves as a high-performance, runtime representation of a static NPC spawn configuration asset (NPCSpawn). It functions as a critical bridge between raw configuration data and the server's live spawning logic.

Its primary architectural purpose is to pre-process and cache spawn rules during server initialization. By converting string-based identifiers (like NPC names) into integer-based indices and pre-calculating complex predicates (like light ranges), it significantly reduces the computational overhead within the main spawning loop.

Each instance of a SpawnWrapper subclass represents a single, complete rule set for spawning a group of related NPCs. The core spawning system maintains a collection of these wrappers and evaluates them against potential spawn locations and world states each tick. In essence, a SpawnWrapper is a stateful predicate object, optimized for frequent evaluation.

## Lifecycle & Ownership

-   **Creation:** SpawnWrapper is an abstract class and cannot be instantiated directly. Concrete subclasses are instantiated by the SpawningPlugin during the server's asset loading phase. A new wrapper is created for each NPCSpawn configuration file that is successfully parsed.

-   **Scope:** An instance of SpawnWrapper persists for the entire lifetime of the server session or until a configuration reload is triggered. They are held in a central registry within the SpawningPlugin.

-   **Destruction:** Instances are eligible for garbage collection when the SpawningPlugin is shut down or when it clears its internal registries during a configuration reload. There is no manual destruction method.

## Internal State & Concurrency

-   **State:** The class is stateful but designed to be **effectively immutable** after construction. The constructor is the sole point of mutation, where it processes the raw NPCSpawn data into optimized lookup maps and predicates. The internal roles map is explicitly wrapped in an unmodifiable view to prevent external changes.

-   **Thread Safety:** The class is **thread-safe for all read operations**. Since its internal state does not change after the constructor completes, its public methods can be safely called from multiple threads simultaneously without locks or synchronization. This design is crucial for any potential future work to parallelize the NPC spawning evaluation across different world regions or threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| spawnParametersMatch(accessor) | boolean | O(1) | Evaluates global world conditions (e.g., time of day, moon phase) against the spawn rule. This is a primary gatekeeping check. |
| shouldDespawn(world, timeManager) | boolean | O(1) | Evaluates global world conditions to determine if an existing NPC governed by this rule should be despawned. |
| withinLightRange(spawningContext) | boolean | O(1) | Performs a highly localized check to determine if a specific block coordinate meets the light level requirements defined in the spawn rule. |
| getSpawnBlockSet(roleIndex) | IntSet | O(1) | Retrieves the pre-computed set of valid block types an NPC can spawn on. Returns null if any block is valid. |
| hasInvalidNPC(name) | boolean | O(1) | Reports whether a specific NPC name from the configuration was found to be invalid during server load. Useful for diagnostics. |

## Integration Patterns

### Standard Usage

The SpawnWrapper is not intended for direct use by game logic or content scripts. It is exclusively consumed by the core Spawning System, which iterates through a collection of wrappers to make spawn decisions.

```java
// Hypothetical usage within the core spawning engine
SpawningContext context = ...; // Context for a potential spawn location
List<SpawnWrapper> allSpawnRules = spawningPlugin.getSpawnRules();

for (SpawnWrapper rule : allSpawnRules) {
    if (rule.spawnParametersMatch(context.accessor) && rule.withinLightRange(context)) {
        // This rule is valid for the current time and location.
        // Proceed with selecting an NPC from rule.getRoles() and attempting to spawn.
        break; 
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never attempt to create instances of SpawnWrapper subclasses manually. The SpawningPlugin manages their lifecycle, ensuring they are correctly initialized with valid indices and registered with the system.
-   **State Modification:** Do not attempt to modify the underlying NPCSpawn object after it has been used to create a SpawnWrapper. The wrapper will not reflect these changes, leading to a desynchronization between configuration and runtime behavior.
-   **Ignoring Initialization Warnings:** The constructor logs warnings for invalid NPC names. Ignoring these warnings can lead to parts of your spawn configuration being silently ignored by the engine. Use the hasInvalidNPC method for health checks.

## Data Pipeline

The SpawnWrapper acts as a filter in the data pipeline that determines NPC spawn eligibility.

> Flow:
> Spawning System Tick → Candidate Block Location → **SpawnWrapper.spawnParametersMatch()** → **SpawnWrapper.withinLightRange()** → Spawn Decision → Entity Creation

