---
description: Architectural reference for CustomModelTexture
---

# CustomModelTexture

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Transient

## Definition
```java
// Signature
public class CustomModelTexture {
```

## Architecture & Concepts
The CustomModelTexture class is a server-side Data Transfer Object (DTO) that represents a single, weighted texture variant for a game model, typically a block. It serves as a direct mapping of configuration data defined in asset files (e.g., JSON or HOCON) into a strongly-typed Java object.

Its primary architectural role is to act as an intermediate representation during the asset loading pipeline. The static **CODEC** field is the most critical feature, defining how raw configuration data with keys like *Texture* and *Weight* is deserialized into a CustomModelTexture instance.

This class forms a bridge between the static, file-based asset configuration system and the dynamic, network-based protocol system. It is not intended for direct use in game logic; instead, its data is transformed into a network-serializable format via the **toPacket** method before being sent to clients.

### Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the Hytale **BuilderCodec** during the server's asset loading phase. The codec reads a block's configuration file and instantiates this class to hold texture variant data. Manual instantiation is rare and discouraged.
- **Scope:** The object's lifetime is bound to its parent asset configuration. It persists in memory as long as the block or model definition that contains it is loaded in the server's asset registry.
- **Destruction:** It is a lightweight object that becomes eligible for garbage collection when the asset registry is reloaded or the server shuts down, releasing the reference to the parent asset.

## Internal State & Concurrency
- **State:** Mutable Plain Old Java Object (POJO). Its state, consisting of a texture path and a weight, is populated by the codec upon creation. While technically mutable, it should be treated as **effectively immutable** after initialization. Modifying its state post-load can lead to desynchronization between the server's configuration and what is sent to clients.
- **Thread Safety:** **Not thread-safe.** This class is a simple data container and lacks any internal synchronization mechanisms. It is designed to be created and processed within the single-threaded context of the server's asset loading pipeline. Concurrent access without external locking will result in undefined behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTexture() | String | O(1) | Retrieves the asset path for the texture. |
| getWeight() | int | O(1) | Retrieves the integer weighting value for this texture variant. |
| toPacket(totalWight) | ModelTexture | O(1) | Transforms this configuration object into a network-ready ModelTexture protocol object. Normalizes the integer weight into a floating-point probability based on the provided total weight. |

## Integration Patterns

### Standard Usage
This class is not meant to be directly manipulated by game logic developers. It is an internal component of the asset system. The system uses its static CODEC to deserialize configuration data automatically.

A conceptual example of the source data that creates a CustomModelTexture instance:

```json
// In a block definition asset file...
"model": {
  "textures": [
    {
      "Texture": "myassets:block/stone_mossy",
      "Weight": 10
    },
    {
      "Texture": "myassets:block/stone_cracked",
      "Weight": 5
    }
  ]
}
```
The asset loader would process this JSON, using the CustomModelTexture.CODEC to create two instances of CustomModelTexture from the *textures* array.

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Modifying the state of a CustomModelTexture instance after it has been loaded by the asset system is a critical anti-pattern. This will cause unpredictable rendering behavior and desynchronization, as the server's internal state will no longer match the original asset files.
- **Direct Instantiation:** Avoid `new CustomModelTexture()`. This class is designed to be populated by the asset pipeline via its static CODEC. Manual creation bypasses validation and the intended data flow.

## Data Pipeline
The CustomModelTexture class is a specific step in the pipeline that transforms static asset definitions into network packets for clients.

> Flow:
> Block Asset File (JSON) -> Hytale Codec System -> **CustomModelTexture** instance -> toPacket() method call -> ModelTexture (Protocol DTO) -> Network Packet Encoder -> Client Renderer

