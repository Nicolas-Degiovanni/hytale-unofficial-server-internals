---
description: Architectural reference for ModelAttachment
---

# ModelAttachment

**Package:** com.hypixel.hytale.server.core.asset.type.model.config
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class ModelAttachment implements NetworkSerializable<com.hypixel.hytale.protocol.ModelAttachment> {
```

## Architecture & Concepts

ModelAttachment is a server-side data structure that defines a visual component to be attached to a parent model, such as a character or entity. It is not a service or manager, but rather a passive data container that represents configuration loaded from an asset file.

The central architectural feature of this class is the static **CODEC** field. This `BuilderCodec` instance makes ModelAttachment a self-describing object, defining its own serialization, deserialization, and validation logic. This pattern tightly couples the data shape with its parsing and validation rules, ensuring that any loaded ModelAttachment instance is guaranteed to be valid according to engine specifications.

By implementing the NetworkSerializable interface, this class serves as a bridge between the server's internal asset representation and the data format required by the client. The `toPacket` method handles the transformation into a network-optimized protocol object, decoupling the server's configuration structure from the client-server communication protocol.

## Lifecycle & Ownership

-   **Creation:** Instances are almost exclusively created by the Hytale asset loading framework during server startup or asset hot-reloading. The framework uses the static CODEC field to deserialize data from configuration files (e.g., JSON) into a new ModelAttachment object. Manual instantiation is strongly discouraged.
-   **Scope:** The lifetime of a ModelAttachment instance is tied to its containing asset. It persists in memory only as long as the parent asset (e.g., a character definition, an item) is loaded. These are transient, lightweight objects.
-   **Destruction:** Instances are managed by the Java Garbage Collector. They are eligible for cleanup once the parent asset is unloaded and all references to the instance are dropped. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency

-   **State:** ModelAttachment is a mutable data container. Its state consists of string identifiers for assets (model, texture, gradient) and a numeric weight. The state is populated once during deserialization and is intended to be treated as read-only thereafter.

-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms. Instances should only be accessed and read from the main server thread. Modifying a ModelAttachment instance from multiple threads will result in unpredictable behavior and data corruption.

## API Surface

The public API is minimal, primarily exposing the configured data and the network transformation contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.ModelAttachment | O(1) | Creates a new network-optimized packet from the instance's state. This is the primary mechanism for sending attachment data to clients. |
| getModel() | String | O(1) | Returns the asset key for the 3D model. |
| getTexture() | String | O(1) | Returns the asset key for the model's texture. |
| getGradientSet() | String | O(1) | Returns the asset key for the gradient set used for colorization. |
| getGradientId() | String | O(1) | Returns the specific ID of the gradient to apply from the gradient set. |
| getWeight() | double | O(1) | Returns the blending weight, typically used for procedural animations or layering. |

## Integration Patterns

### Standard Usage

Developers do not typically interact with the ModelAttachment Java class directly. Instead, they define its data within a higher-level asset configuration file. The engine's asset loader then uses the ModelAttachment.CODEC to parse this data into an object at runtime.

A conceptual asset file might look like this:

```json
// in some_character.json
{
  "name": "ExampleCharacter",
  "attachments": [
    {
      "Model": "hytale:character/attachment/helmet_01",
      "Texture": "hytale:character/attachment/helmet_01_texture_blue",
      "GradientSet": "hytale:cosmetics/gradients/common_metals",
      "GradientId": "steel",
      "Weight": 1.0
    }
  ]
}
```

The server would then access the loaded data from the parent object.

```java
// Example of how the system might access the loaded object
CharacterAsset character = assetManager.load("some_character.json");
ModelAttachment helmet = character.getAttachments().get(0);
com.hypixel.hytale.protocol.ModelAttachment packet = helmet.toPacket();
player.sendPacket(packet);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new ModelAttachment()`. The object's integrity relies on the validation logic embedded within its CODEC. Bypassing the codec can lead to invalid or incomplete objects that will cause runtime errors downstream.
-   **Post-Load Modification:** Do not modify the state of a ModelAttachment instance after it has been loaded by the asset system. The engine may cache or share these instances, and runtime mutations can lead to inconsistent state across the server. Treat them as effectively immutable after creation.
-   **Concurrent Access:** Do not share a ModelAttachment instance across multiple threads without external synchronization. The object is not designed for concurrent access.

## Data Pipeline

The ModelAttachment class is a key component in the pipeline that transforms static configuration on disk into live data sent to the game client.

> Flow:
> Asset File (JSON) -> Asset Loading Service -> **ModelAttachment.CODEC** -> **ModelAttachment Instance** -> `toPacket()` -> Network Protocol Layer -> Game Client
---

