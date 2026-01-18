---
description: Architectural reference for StashGameplayConfig
---

# StashGameplayConfig

**Package:** com.hypixel.hytale.builtin.adventure.stash
**Type:** Configuration Model

## Definition
```java
// Signature
public class StashGameplayConfig {
```

## Architecture & Concepts
StashGameplayConfig is a strongly-typed data model that represents a specific slice of gameplay configuration for the "Stash" adventure mode feature. It is not a service or manager; its sole purpose is to provide a type-safe, in-memory representation of settings defined in external game asset files, typically JSON.

The core of its design is the static **CODEC** field. The Hytale engine's configuration loading system uses this codec to deserialize raw configuration data into a hydrated StashGameplayConfig instance. This pattern ensures that configuration is validated and structured at load time, preventing runtime errors from malformed data.

This class is designed as a plugin or sub-component to the primary GameplayConfig object. By namespacing its settings within this class, the system avoids configuration key collisions and logically groups related parameters. The codec's inheritance support allows for layering configurations, where a base configuration can be overridden by more specific contexts (e.g., world-specific settings overriding global defaults).

## Lifecycle & Ownership
- **Creation:** Instances are not intended for direct developer instantiation. They are materialized exclusively by the Hytale asset and configuration system during server or world bootstrap. The static CODEC is invoked by the framework to construct the object from a configuration source. A static, default-constructed instance exists as a fallback mechanism.

- **Scope:** The lifetime of a StashGameplayConfig instance is strictly bound to the lifetime of its parent GameplayConfig object. It persists as long as that gameplay context is active and is immutable for that duration.

- **Destruction:** The object is marked for garbage collection when its parent GameplayConfig is discarded. This typically occurs during a server shutdown, world unload, or context switch.

## Internal State & Concurrency
- **State:** The internal state is mutable only during the deserialization process managed by the CODEC. After initialization, the object should be treated as **effectively immutable**. Its state consists of simple primitive fields representing configuration values.

- **Thread Safety:** This class is **not thread-safe** and contains no internal synchronization. However, it is safe for concurrent use under the engine's standard operating model: configuration is loaded and written once on a main thread during startup, and subsequently read by multiple game logic threads.

   **WARNING:** Any external attempt to mutate the state of a StashGameplayConfig instance after it has been loaded is a severe anti-pattern and will lead to unpredictable behavior and race conditions.

## API Surface
The public API is minimal, focusing on static accessors to retrieve the configuration from a broader context.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(GameplayConfig config) | static StashGameplayConfig | O(1) | Retrieves the Stash-specific config from a parent GameplayConfig. May return null if not defined. |
| getOrDefault(GameplayConfig config) | static StashGameplayConfig | O(1) | Null-safe version of get. Returns a static default instance if the configuration is not present. |
| isClearContainerDropList() | boolean | O(1) | Accessor for the primary configuration value. |

## Integration Patterns

### Standard Usage
The correct pattern is to retrieve the Stash configuration from an existing GameplayConfig context, using the null-safe default provider to ensure system stability.

```java
// In a system where 'gameplayConfig' is an available resource
StashGameplayConfig stashConfig = StashGameplayConfig.getOrDefault(gameplayConfig);
boolean shouldClear = stashConfig.isClearContainerDropList();

if (shouldClear) {
    // Execute game logic based on the configuration
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new StashGameplayConfig()`. This bypasses the entire configuration loading pipeline and will result in an object with default, not user-configured, values.

- **State Mutation:** Do not use reflection or other means to modify the fields of this object at runtime. Configuration is designed to be read-only once loaded.

## Data Pipeline
StashGameplayConfig is the terminal point in a configuration data pipeline. It translates raw data from disk into a type-safe object for consumption by game logic.

> Flow:
> Game Asset (e.g., `config.json`) -> Engine Asset Loader -> GameplayConfig Deserializer -> **StashGameplayConfig CODEC** -> In-Memory StashGameplayConfig Instance -> Game Systems

