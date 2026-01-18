---
description: Architectural reference for PortalDeviceConfig
---

# PortalDeviceConfig

**Package:** com.hypixel.hytale.builtin.portals.components
**Type:** Configuration Model

## Definition
```java
// Signature
public class PortalDeviceConfig {
```

## Architecture & Concepts
The PortalDeviceConfig class is a pure data model that represents the deserialized configuration for a portal device. It is not a service or a manager; its sole responsibility is to act as a strongly-typed container for properties defined in game asset files, typically JSON.

The key architectural feature is the static **CODEC** field. This `BuilderCodec` instance defines the data schema and deserialization logic. The Hytale engine's asset loading system uses this codec to automatically parse the corresponding data from an asset file and instantiate a PortalDeviceConfig object.

This class serves as the immutable contract between content creators (who define portal behavior in data files) and the portal runtime systems. The portal logic reads from this configuration object to determine which block states to render for the device's `on`, `off`, and `spawning` phases, and which block to place upon arrival in a new world.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the engine's `Codec` and asset loading systems during server or client startup. A developer **must not** instantiate this class directly using the `new` keyword. It is constructed when a component that contains a PortalDeviceConfig is loaded from a data file.
-   **Scope:** The lifetime of a PortalDeviceConfig instance is tied to the lifetime of its owning asset, such as a `BlockType` or an entity prefab. It persists in memory as long as the parent asset is loaded.
-   **Destruction:** The object is marked for garbage collection when its parent asset is unloaded from memory, for example, when a world is shut down or game assets are reloaded.

## Internal State & Concurrency
-   **State:** The state of this object is **effectively immutable**. After being instantiated and populated by the `Codec` system, its properties are not intended to be changed. There are no public setters, and all fields are treated as read-only configuration data.
-   **Thread Safety:** This class is **thread-safe for reads**. Due to its immutable nature post-creation, multiple systems (e.g., game logic, rendering) can safely access its properties concurrently without requiring locks or other synchronization primitives. Writing to this object is not a supported operation and would violate its design contract.

## API Surface
The public API is designed for read-only access to the portal device's configuration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getOnState() | String | O(1) | Returns the state key for when the portal is active. |
| getSpawningState() | String | O(1) | Returns the state key for the portal's creation/transition animation. |
| getOffState() | String | O(1) | Returns the state key for when the portal device is inactive. |
| getReturnBlock() | String | O(1) | Returns the asset name of the block to place at the destination. May be null. |
| areBlockStatesValid(BlockType) | boolean | O(1) | Validates that all configured state keys exist on the provided base BlockType. |

## Integration Patterns

### Standard Usage
This component is typically retrieved from a `BlockType` or an entity's component data. The primary use case is to read its state keys and use them to manipulate the world or validate configuration.

```java
// Example: Validating a portal block type during server startup
void validatePortalDevice(BlockType portalBlock) {
    PortalDeviceConfig config = portalBlock.getComponent(PortalDeviceConfig.class);

    if (config == null) {
        // This block is not a configured portal device
        return;
    }

    if (!config.areBlockStatesValid(portalBlock)) {
        // CRITICAL: This is a content error. The JSON configuration for this
        // block refers to block states that do not exist.
        throw new IllegalStateException("Invalid portal device configuration for: " + portalBlock.getName());
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance with `new PortalDeviceConfig()`. This bypasses the asset loading system and results in an object with default, unconfigured values. Always retrieve it from an existing entity or asset.
-   **State Mutation:** Do not attempt to modify the state of this object via reflection or other means. The portal systems rely on this configuration being a stable, read-only snapshot of the asset data. Modifying it at runtime can lead to desynchronization and undefined behavior.

## Data Pipeline
PortalDeviceConfig is not a processing node in a pipeline; rather, it is the deserialized result of one. The data flows from a static asset file into an in-memory instance of this class.

> Flow:
> Game Asset File (e.g., `portal_device.json`) -> Hytale Asset Loader -> **PortalDeviceConfig.CODEC** -> **PortalDeviceConfig Instance** -> Portal System Logic

