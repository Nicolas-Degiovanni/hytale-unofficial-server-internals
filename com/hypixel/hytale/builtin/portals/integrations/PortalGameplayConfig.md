---
description: Architectural reference for PortalGameplayConfig
---

# PortalGameplayConfig

**Package:** com.hypixel.hytale.builtin.portals.integrations
**Type:** Data Structure

## Definition
```java
// Signature
public class PortalGameplayConfig {
```

## Architecture & Concepts
The PortalGameplayConfig class is a data structure, not a service. Its sole responsibility is to act as a strongly-typed container for configuration values related to portal gameplay mechanics. It is designed to be populated directly from asset files (e.g., JSON) via Hytale's serialization framework.

The static **CODEC** field is the most critical part of this class. It defines the contract for how data from an external source is mapped into a Java object instance. This pattern decouples the gameplay logic that *uses* the configuration from the data format that *stores* it. This class aggregates various sub-configurations, such as the VoidEventConfig, providing a single, coherent object for systems that manage portal behavior.

This component exists at the data layer and is considered a passive model object. It contains no business logic.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale codec system during the asset loading phase. The public static **CODEC** field is used by a higher-level manager to deserialize a configuration file into a PortalGameplayConfig object. Manual instantiation is an anti-pattern.
- **Scope:** The lifetime of a PortalGameplayConfig instance is tied to the scope of the system that loaded it. For example, if loaded as part of a world's definition, it will persist as long as that world is active in memory. It is not a global or session-scoped object.
- **Destruction:** The object is eligible for garbage collection once the owning system (e.g., a WorldManager or ZoneController) is unloaded and releases its reference. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The internal state is mutable upon creation. The codec system populates the private fields after constructing the object. However, after deserialization is complete, the object should be treated as **effectively immutable**. Modifying its state post-initialization can lead to inconsistent behavior across the engine.
- **Thread Safety:** This class is **not thread-safe**. It is a simple data container with no internal synchronization mechanisms. It is expected to be deserialized, passed to the main game thread, and then read from that thread only. Any cross-thread access must be managed externally with proper synchronization.

## API Surface
The public API is minimal, exposing only read-only access to the contained configuration data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getVoidEvent() | VoidEventConfig | O(1) | Retrieves the configuration for the "Void Event" mechanic. May return null if not defined in the source asset. |

## Integration Patterns

### Standard Usage
The intended pattern is to load this configuration via a central asset or configuration manager and pass the resulting object to the relevant gameplay systems.

```java
// Concept: Loading and using the configuration within a hypothetical PortalSystem
// Note: The actual loading mechanism is abstracted by the engine.

// 1. The engine's config loader uses the static CODEC to deserialize the object
PortalGameplayConfig config = assetLoader.load("my_portal_config.json", PortalGameplayConfig.CODEC);

// 2. A gameplay system retrieves the specific sub-configuration
VoidEventConfig voidConfig = config.getVoidEvent();
if (voidConfig != null) {
    this.voidEventManager.configure(voidConfig);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PortalGameplayConfig()`. This creates an empty object with null fields, which will cause NullPointerExceptions in any system that consumes it. The object is invalid unless populated by the codec.
- **Post-Load Mutation:** Do not attempt to modify the state of a PortalGameplayConfig object after it has been loaded and shared. This violates the assumption that configuration is static for its lifetime and can lead to race conditions or unpredictable gameplay behavior.

## Data Pipeline
PortalGameplayConfig serves as the destination for a data deserialization pipeline. It translates raw data from disk into a type-safe object for consumption by the game engine.

> Flow:
> Asset File (e.g., JSON) -> Hytale Codec Engine -> **PortalGameplayConfig** -> Portal Gameplay System

