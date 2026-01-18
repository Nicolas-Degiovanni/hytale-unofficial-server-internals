---
description: Architectural reference for InstanceWorldConfig
---

# InstanceWorldConfig

**Package:** com.hypixel.hytale.builtin.instances.config
**Type:** Configuration Data Object

## Definition
```java
// Signature
public class InstanceWorldConfig {
```

## Architecture & Concepts
The **InstanceWorldConfig** class is not a service or manager, but a structured data container that defines the behavior of a game world designated as an "Instance". Instances are typically temporary, dynamically-created worlds for activities like dungeons, minigames, or quests.

This class acts as a schema for a specific section within a world's primary configuration file. The Hytale server's world-loading mechanism uses the static **CODEC** field, a powerful serialization and deserialization engine, to parse the relevant configuration data and instantiate this object.

Once loaded, an **InstanceWorldConfig** object is attached to the main **WorldConfig** for that world. It does not actively *do* anything on its own; rather, higher-level systems query this configuration object to make decisions. For example, the **InstanceManager** might check the **RemovalConditions** to determine if an empty instance should be shut down, and the **PlayerConnectionManager** might check **preventReconnection** to decide where to place a returning player.

## Lifecycle & Ownership
The lifecycle of an **InstanceWorldConfig** is strictly bound to the lifecycle of the world it configures.

-   **Creation:** This object is almost never instantiated directly using its constructor. It is created by the Hytale **Codec** system during the world loading process. The system reads configuration from a data source (e.g., a JSON file), finds the "Instance" section, and uses the **CODEC** definition to build the object. Alternatively, it can be created and attached programmatically via the static **ensureAndGet** method, which is useful for dynamically converting a standard world into an instance at runtime.
-   **Scope:** The object exists for the entire duration that its parent **WorldConfig** is held in memory. Its scope is identical to the loaded world's scope.
-   **Destruction:** The object is marked for garbage collection when the world is unloaded and its parent **WorldConfig** is dereferenced. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** The internal state is fully mutable. It is a plain data object whose properties (like **returnPoint** or **removalConditions**) are intended to be read and potentially modified by server systems during the world's lifetime. It holds no caches or derived data.
-   **Thread Safety:** This class is **not thread-safe**. It contains no locks, volatile keywords, or other concurrency primitives.

    **WARNING:** All reads and writes to an **InstanceWorldConfig** instance must be synchronized externally or confined to the main server thread that owns the corresponding world. Unsynchronized access from multiple threads (e.g., a network thread and the main game loop thread) will result in race conditions, memory visibility issues, and severe, difficult-to-diagnose bugs.

## API Surface
The primary API consists of static factory methods and standard getters/setters.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(WorldConfig) | static InstanceWorldConfig | O(1) | Retrieves the attached instance config from a world config, returning null if not present. |
| ensureAndGet(WorldConfig) | static InstanceWorldConfig | O(1) | Retrieves the attached instance config, or creates and attaches a new default one if not present. |
| getRemovalConditions() | RemovalCondition[] | O(1) | Returns the array of conditions that trigger automatic world removal. |
| setRemovalConditions(RemovalCondition...) | void | O(1) | Overwrites the set of removal conditions for the instance. |
| shouldPreventReconnection() | boolean | O(1) | Returns true if players are blocked from logging directly back into this world. |
| getReturnPoint() | WorldReturnPoint | O(1) | Returns the location where players are sent upon leaving this instance. |

## Integration Patterns

### Standard Usage
The correct pattern is to retrieve the configuration from an existing **WorldConfig** object, preferably using the safe **ensureAndGet** method to avoid null pointer exceptions.

```java
// In a system that has access to a World object
WorldConfig worldConfig = world.getConfig();
InstanceWorldConfig instanceConfig = InstanceWorldConfig.ensureAndGet(worldConfig);

// Now, read or modify the configuration
if (instanceConfig.shouldPreventReconnection()) {
    // Logic to handle player redirection
}

// Dynamically set a return point
WorldReturnPoint spawn = new WorldReturnPoint("world_lobby");
instanceConfig.setReturnPoint(spawn);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance with `new InstanceWorldConfig()`. This creates an "orphan" configuration object that is not associated with any world. The rest of the engine will be unaware of its existence, and any settings applied to it will have no effect.
-   **State Caching:** Do not fetch the **InstanceWorldConfig** once and store it long-term in another service. The configuration can potentially be modified at runtime. Always fetch it from the authoritative **WorldConfig** source when you need to make a decision based on its values.

## Data Pipeline
The **InstanceWorldConfig** primarily participates in the configuration loading pipeline, not a real-time data processing pipeline.

> Flow:
> World Configuration File (e.g., world.json) -> Hytale Codec Deserializer -> **InstanceWorldConfig** (Object in Memory) -> Attached to WorldConfig -> Read by Server Systems (e.g., InstanceManager)

