---
description: Architectural reference for CreativeHubWorldConfig
---

# CreativeHubWorldConfig

**Package:** com.hypixel.hytale.builtin.creativehub.config
**Type:** Configuration Model

## Definition
```java
// Signature
public class CreativeHubWorldConfig {
```

## Architecture & Concepts
CreativeHubWorldConfig is a passive data structure that encapsulates world-specific configuration for the "CreativeHub" game mode. It is not a service or a manager; its sole responsibility is to act as a type-safe container for settings deserialized from a world's configuration files.

The key architectural feature is the static **CODEC** field. This self-describing `BuilderCodec` integrates the class with Hytale's core configuration loading system. It allows the engine to automatically instantiate and populate this object from a data source, such as a JSON file, without requiring custom parsing logic. This class exemplifies a plugin-based configuration pattern, where a generic `WorldConfig` object is extended with specialized, strongly-typed data for different modules or game modes.

## Lifecycle & Ownership
- **Creation:** This object is instantiated exclusively by the Hytale `Codec` framework during the server's world loading sequence. The `BuilderCodec` uses the provided constructor reference (`CreativeHubWorldConfig::new`) to create a new instance, which is then populated with data from the configuration source. Direct instantiation by developers is an anti-pattern.

- **Scope:** The lifecycle of a CreativeHubWorldConfig instance is tightly bound to the parent `WorldConfig` object that contains it. It persists in memory for as long as the world itself is loaded on the server.

- **Destruction:** The object is eligible for garbage collection when its parent `WorldConfig` is unloaded, typically during a world change or server shutdown. It does not manage any unmanaged resources and has no explicit destruction method.

## Internal State & Concurrency
- **State:** The internal state is mutable upon creation but should be treated as **effectively immutable** after deserialization. It holds configuration data that is read at startup and is not expected to change during runtime. It does not cache any data beyond the initial values loaded from the configuration file.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization mechanisms. This design is intentional and safe under the assumption that the object is populated once on a single thread during server initialization and subsequently only read by other systems.

**WARNING:** Modifying the state of a retrieved CreativeHubWorldConfig instance after the world has loaded is unsupported and will lead to unpredictable behavior, as different systems may read the configuration at different times.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ID | static String | O(1) | A constant "CreativeHub" used as the registration key within the plugin configuration system. |
| CODEC | static BuilderCodec | N/A | The self-describing codec used by the engine to serialize and deserialize this object. |
| get(WorldConfig) | static CreativeHubWorldConfig | O(1) | The primary factory method for retrieving the specialized config from a generic WorldConfig instance. |
| getStartupInstance() | String | O(1) | Returns the name of the initial instance players should spawn into. May be null. |

## Integration Patterns

### Standard Usage
The intended pattern is to retrieve this specialized configuration from a core `WorldConfig` object, which is typically available in world-scoped contexts.

```java
// Assume 'worldConfig' is an existing, loaded WorldConfig object
CreativeHubWorldConfig hubConfig = CreativeHubWorldConfig.get(worldConfig);

if (hubConfig != null) {
    String instance = hubConfig.getStartupInstance();
    if (instance != null) {
        // Logic to handle the startup instance
        spawnPlayerIn(instance);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new CreativeHubWorldConfig()`. This bypasses the configuration loading pipeline, resulting in an empty and non-functional object. The only valid way to obtain an instance is via the static `get` method.

- **Runtime Modification:** Do not attempt to modify the state of this object after it has been retrieved. Configuration is designed to be a read-only snapshot of the world's settings at load time.

## Data Pipeline
This class sits at the end of the configuration loading pipeline. It translates raw configuration data into a strongly-typed, in-memory Java object.

> Flow:
> World Configuration File (e.g., world.json) -> Server Boot Process -> WorldConfig Loader -> **CreativeHubWorldConfig.CODEC** -> **CreativeHubWorldConfig Instance** (stored in WorldConfig) -> Game Systems (via `get()` method)

