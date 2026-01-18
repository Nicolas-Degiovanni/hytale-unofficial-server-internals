---
description: Architectural reference for ItemPullbackConfig
---

# ItemPullbackConfig

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Transient Data Model

## Definition
```java
// Signature
public class ItemPullbackConfig implements NetworkSerializable<ItemPullbackConfiguration> {
```

## Architecture & Concepts
ItemPullbackConfig is a server-side data model that encapsulates configuration for item animation overrides. Its primary function is to define custom positional and rotational offsets for an item held in a character's hands during a "pullback" animation, such as drawing a bow or charging a weapon.

This class serves as a critical data bridge between two distinct systems:

1.  **The Asset System:** It is designed to be deserialized from on-disk asset files (e.g., JSON) using its static CODEC field. This allows game designers to define animation properties declaratively. The codec also handles the important translation from high-precision Vector3d types used in configuration files to the more network-efficient Vector3f type used at runtime.
2.  **The Network Protocol:** By implementing NetworkSerializable, this class can be converted into a network-ready packet object (ItemPullbackConfiguration). This configuration is then sent to the client, which uses the data to adjust the client-side rendering of the item in the player's hands.

It is not a service or a manager; it is a plain data-holding object that is part of a larger item definition.

### Lifecycle & Ownership
-   **Creation:** Instances are almost exclusively created by the asset loading system during server bootstrap or asset hot-reloading. The system uses the public static CODEC to deserialize the configuration from an item's asset definition file. The object is owned by the parent item configuration object.
-   **Scope:** The lifecycle of an ItemPullbackConfig instance is bound to the lifecycle of its parent item asset. It is loaded into memory once and persists as part of the server's static asset registry for the entire server session.
-   **Destruction:** The object is marked for garbage collection only when the asset registry is cleared, which typically happens during a full server shutdown.

## Internal State & Concurrency
-   **State:** The internal state consists of four nullable Vector3f fields representing offsets and rotations. While the fields are not final, the object should be treated as **Effectively Immutable** after its initial creation by the asset loader. Modifying this object at runtime is a severe anti-pattern that can lead to state desynchronization.
-   **Thread Safety:** This class is **Not Thread-Safe**. It contains no internal locking or synchronization primitives. All deserialization and registration must be performed by the single-threaded asset loading system. Once loaded and safely published to the asset registry, it can be read by multiple threads, but any mutation is unsafe.

## API Surface
The public contract is minimal, focusing on serialization and data transfer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | - | A static codec used by the asset system to deserialize this object from configuration files. |
| toPacket() | ItemPullbackConfiguration | O(1) | Serializes the object into a network packet for client transmission. This is the primary runtime operation. |

## Integration Patterns

### Standard Usage
This class is not intended for direct manipulation by game logic developers. Its usage is implicit and handled by the engine's asset and networking systems. A game designer defines the values in a corresponding item JSON file.

```json
// Example Item Asset (conceptual)
{
  "name": "composite_bow",
  "pullbackConfig": {
    "LeftOffsetOverride": [0.1, 0.0, -0.2],
    "LeftRotationOverride": [0.0, 90.0, 0.0],
    "RightOffsetOverride": [0.0, 0.0, 0.0],
    "RightRotationOverride": [0.0, 0.0, 0.0]
  }
}
```
The engine automatically deserializes the `pullbackConfig` block into an ItemPullbackConfig instance.

### Anti-Patterns (Do NOT do this)
-   **Runtime Modification:** Do not retrieve an item's configuration from the asset registry and attempt to change its pullback values at runtime. This will cause unpredictable behavior and is not thread-safe. Configuration is meant to be static and loaded from source assets.
-   **Direct Instantiation:** Avoid using `new ItemPullbackConfig()` in game logic. This object's state should be sourced from asset files to ensure consistency and maintainability.

## Data Pipeline
The primary flow for this data is from a configuration file on disk to a network packet sent to the game client.

> Flow:
> Item Asset File (JSON) -> Asset Loading Service -> **ItemPullbackConfig.CODEC** -> In-Memory **ItemPullbackConfig** Instance -> `toPacket()` -> `ItemPullbackConfiguration` Packet -> Network Layer -> Game Client for Rendering

