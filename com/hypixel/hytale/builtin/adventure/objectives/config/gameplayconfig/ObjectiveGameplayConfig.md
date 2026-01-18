---
description: Architectural reference for ObjectiveGameplayConfig
---

# ObjectiveGameplayConfig

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.gameplayconfig
**Type:** Data Object / Configuration

## Definition
```java
// Signature
public class ObjectiveGameplayConfig {
```

## Architecture & Concepts
ObjectiveGameplayConfig is a strongly-typed data object that represents a specific, structured block of configuration within a larger GameplayConfig asset. It is not a service or manager; its sole purpose is to act as a data contract for settings related to the adventure mode objective system.

The core of this class is the static **CODEC** field. This `BuilderCodec` instance is a declarative definition of how to serialize and deserialize the ObjectiveGameplayConfig from a data source, such as a world's JSON configuration file. The engine's asset loading and codec systems use this definition to automatically instantiate and populate this object.

The static **ID** field, "Objective", serves as the key to locate this configuration block within its parent GameplayConfig plugin map. This mechanism allows for a modular and extensible configuration system where different game features can define their own data structures without interfering with others.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale codec system during the deserialization of a parent GameplayConfig asset. The constructor is invoked by the `BuilderCodec`, which then populates the `starterObjectiveLinePerWorld` field from the source data. Direct manual instantiation is an anti-pattern.
- **Scope:** The lifecycle of an ObjectiveGameplayConfig instance is strictly bound to its containing GameplayConfig object. It persists in memory as long as the parent configuration is required, typically for the duration of a game session in a specific world.
- **Destruction:** The object is marked for garbage collection when its parent GameplayConfig is unloaded or otherwise goes out of scope. It has no explicit cleanup or `close` method.

## Internal State & Concurrency
- **State:** The object's state is effectively immutable after its creation by the codec. The `starterObjectiveLinePerWorld` map is populated once during deserialization and is not intended to be modified thereafter. There are no public methods to mutate the internal state.
- **Thread Safety:** This class is not inherently thread-safe. However, due to its effectively immutable nature post-creation, it can be safely read by multiple threads. All public methods are safe for concurrent access, assuming the parent GameplayConfig object from which it is retrieved is managed in a thread-safe manner.

**WARNING:** The Map returned by `getStarterObjectiveLinePerWorld` is a direct reference to the internal state. Modifying this map from any thread is a severe anti-pattern that will lead to unpredictable behavior and violates the class's design contract.

## API Surface
The public API is minimal, focusing on retrieval of the configuration data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(GameplayConfig config) | static ObjectiveGameplayConfig | O(1) | Retrieves the objective-specific configuration from a generic GameplayConfig container. Returns null if not present. |
| getStarterObjectiveLinePerWorld() | Map<String, String> | O(1) | Returns the map of world names to their corresponding starting objective line asset identifiers. |

## Integration Patterns

### Standard Usage
The intended pattern is to retrieve this specialized configuration from a loaded, general-purpose GameplayConfig object. This is typically done by systems that need to initialize or manage game objectives.

```java
// Assume 'gameplayConfig' is a fully loaded object for the current world
ObjectiveGameplayConfig objectiveConfig = ObjectiveGameplayConfig.get(gameplayConfig);

if (objectiveConfig != null) {
    Map<String, String> starterObjectives = objectiveConfig.getStarterObjectiveLinePerWorld();
    String currentWorldObjective = starterObjectives.get("world_zone_1");

    // Use the retrieved objective identifier to initialize the player's quest line
    ObjectiveSystem.startObjectiveLine(player, currentWorldObjective);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new ObjectiveGameplayConfig()`. This creates an empty, useless object. The only valid instances are those created by the engine's configuration loader via the defined CODEC.
- **State Mutation:** Do not modify the map returned by `getStarterObjectiveLinePerWorld()`. This breaks the immutability contract and can cause state corruption across different systems that read the same configuration object.

## Data Pipeline
This class sits at the end of the configuration loading pipeline, serving as a structured, in-memory representation of on-disk data.

> Flow:
> World Asset File (e.g., game.json) -> Engine Asset Loader -> `GameplayConfig` Deserialization -> **ObjectiveGameplayConfig** (populated via CODEC) -> Objective System Logic

