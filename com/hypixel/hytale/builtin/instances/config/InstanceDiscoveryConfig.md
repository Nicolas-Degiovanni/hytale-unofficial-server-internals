---
description: Architectural reference for InstanceDiscoveryConfig
---

# InstanceDiscoveryConfig

**Package:** com.hypixel.hytale.builtin.instances.config
**Type:** Configuration Model / DTO

## Definition
```java
// Signature
public class InstanceDiscoveryConfig {
```

## Architecture & Concepts
The InstanceDiscoveryConfig class is a passive data structure that defines the audio-visual feedback presented to a player upon discovering a new game instance, such as a dungeon or a point of interest. It is not a service or manager, but rather a data-driven configuration model.

The most critical architectural feature is the static **CODEC** field. This `BuilderCodec` provides a declarative schema for serialization and deserialization. The engine's asset pipeline uses this codec to parse configuration files (e.g., JSON) and hydrate them into strongly-typed InstanceDiscoveryConfig objects. This decouples the game's presentation logic from hard-coded values, allowing designers to define and tune discovery events entirely through data files.

This class serves as the data contract between an instance's definition and the client-side UI and audio playback systems.

## Lifecycle & Ownership
- **Creation:** Instances of this class are not intended to be created directly with the `new` keyword in game logic. They are instantiated by the Hytale Codec framework during the asset loading process when the engine parses an instance's configuration file.
- **Scope:** The lifetime of an InstanceDiscoveryConfig object is typically bound to the lifetime of the in-memory representation of the game instance it belongs to. It persists as long as the parent asset is loaded.
- **Destruction:** The object is marked for garbage collection when its parent instance asset is unloaded from memory. It does not manage any native resources and has no explicit destruction method.

## Internal State & Concurrency
- **State:** The object is fully mutable. It is a container for configuration properties with public setters for each field. The presence of a `clone` method indicates that these configurations may be used as templates and modified at runtime for specific scenarios.

- **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization primitives. It is designed to be loaded on a main or asset-loading thread and subsequently read by game systems.

    **Warning:** Modifying a shared InstanceDiscoveryConfig object from multiple threads will lead to race conditions and unpredictable behavior. If mutation is required at runtime, always operate on a cloned instance.

## API Surface
The primary public contract consists of standard Java Bean-style getters and setters for its configuration properties. The following method is of architectural significance.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| clone() | InstanceDiscoveryConfig | O(1) | Creates a new instance of the object with a shallow copy of all field values. Essential for runtime modifications. |

## Integration Patterns

### Standard Usage
The intended pattern is to retrieve a pre-loaded configuration from a parent game object, such as an Instance definition, and pass it to the relevant client systems.

```java
// Example: A system that handles instance discovery events
void onPlayerDiscoverInstance(Player player, Instance instance) {
    InstanceDiscoveryConfig config = instance.getDiscoveryConfig();

    if (config != null && config.isDisplay()) {
        // Pass the configuration to the UI and audio managers
        uiManager.showDiscoveryTitle(config);
        audioManager.playSound(config.getDiscoverySoundEventId());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new InstanceDiscoveryConfig()`. This creates an object with default field values, completely bypassing the data-driven configuration loaded from asset files. The resulting object will not reflect the designer's intent.

- **Shared State Mutation:** Modifying a globally-loaded configuration object is extremely dangerous. This can cause visual or audio artifacts for all subsequent discovery events that reference the same configuration. If a temporary modification is needed, use the `clone` method first.
    ```java
    // BAD: Modifying a shared config
    InstanceDiscoveryConfig sharedConfig = instance.getDiscoveryConfig();
    sharedConfig.setDuration(10.0f); // This affects all future uses of this config

    // GOOD: Modifying a clone
    InstanceDiscoveryConfig localConfig = instance.getDiscoveryConfig().clone();
    localConfig.setDuration(10.0f); // This change is isolated
    uiManager.showDiscoveryTitle(localConfig);
    ```

## Data Pipeline
InstanceDiscoveryConfig sits at the end of the asset loading pipeline and at the beginning of the client-side feedback pipeline. It translates static data from files into actionable parameters for engine systems.

> Flow:
> Game Asset File (e.g., JSON) -> Asset Loading System -> **BuilderCodec Deserializer** -> **InstanceDiscoveryConfig Object** -> Game Logic (Player Event) -> UI & Audio Systems

