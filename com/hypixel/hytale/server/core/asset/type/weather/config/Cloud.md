---
description: Architectural reference for Cloud
---

# Cloud

**Package:** com.hypixel.hytale.server.core.asset.type.weather.config
**Type:** Data Model / Configuration Object

## Definition
```java
// Signature
public class Cloud implements NetworkSerializable<com.hypixel.hytale.protocol.Cloud> {
```

## Architecture & Concepts

The Cloud class is a server-side data model that represents a single, distinct layer of clouds within a broader weather profile asset. It is not a live, in-world entity but rather a static configuration object that defines the visual properties and behavior of a cloud type.

Its primary architectural function is to serve as a deserialization target for the Hytale asset system. The static **CODEC** field is the cornerstone of this class, defining how a JSON object from a weather asset file is parsed, validated, and transformed into a Java Cloud instance.

Crucially, this class implements the NetworkSerializable interface. This designates it as a bridge between the server's internal asset configuration and the data sent to clients. The server loads a Cloud object from a file, and when it needs to inform a client about the current weather, it calls the **toPacket** method to generate a network-optimized `com.hypixel.hytale.protocol.Cloud` object for transmission.

The presence of UIEditor metadata within the CODEC definition strongly indicates that this class's properties are designed to be configured visually within Hytale's development tools, with fields like *colors* and *speeds* being represented as editable timelines.

## Lifecycle & Ownership

-   **Creation:** A Cloud instance is never created directly with the *new* keyword in game logic. It is instantiated exclusively by the Hytale **Codec** system when a parent weather profile asset (e.g., `sunny.json`) is loaded by the server's AssetManager. The `BuilderCodec` uses the protected no-argument constructor for this purpose.
-   **Scope:** The lifetime of a Cloud object is strictly bound to its parent weather profile asset. It is loaded into memory when the profile becomes active for a world or biome and persists as long as that profile is referenced.
-   **Destruction:** The object is marked for garbage collection when its parent weather profile is unloaded from memory. It has no explicit destruction or cleanup methods.

## Internal State & Concurrency

-   **State:** The object's state is mutable upon creation to allow the `BuilderCodec` to populate its fields. However, after asset loading is complete, it should be treated as an **immutable** configuration object. Modifying its fields at runtime is an unsupported operation that can lead to desynchronization between the server's state and what clients perceive.
-   **Thread Safety:** This class is **not thread-safe**. It is a simple data container. It is designed to be created and populated on an asset loading thread and subsequently read by the main server thread. Any concurrent read/write access must be synchronized externally, though such a pattern is highly discouraged.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.Cloud | O(N+M) | Transforms this server-side configuration object into a network-ready packet for client transmission. N is the length of the colors array, M is the length of the speeds array. |
| getTexture() | String | O(1) | Returns the asset path for the cloud layer's texture. |
| getColors() | TimeColorAlpha[] | O(1) | Returns the raw array of time-keyed color and alpha values. |
| getSpeeds() | TimeFloat[] | O(1) | Returns the raw array of time-keyed cloud movement speeds. |

## Integration Patterns

### Standard Usage

A developer or content creator does not interact with this class in Java code. The standard pattern is to define its properties declaratively within a weather profile JSON asset. The engine handles the loading and lifecycle.

```json
// Example from a hypothetical weather.json asset
{
  "clouds": [
    {
      "Texture": "hytale:sky/clouds/default_clouds",
      "Colors": [
        { "time": 0.0, "value": "FFFFFF", "alpha": 0.8 },
        { "time": 0.5, "value": "F0F0F0", "alpha": 0.9 },
        { "time": 1.0, "value": "FFFFFF", "alpha": 0.8 }
      ],
      "Speeds": [
        { "time": 0.0, "value": 1.5 },
        { "time": 1.0, "value": 1.5 }
      ]
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new Cloud()`. The object is fundamentally incomplete without being populated by the asset loader via its CODEC. This will result in null fields and produce `NullPointerException`s when `toPacket` is called.
-   **Runtime Modification:** Do not get the color or speed arrays and modify their contents after an asset has been loaded. This breaks the "configuration as code" paradigm and will not be automatically synchronized to clients.
-   **Stateful Caching:** Do not cache the result of `toPacket()`. The transformation may be extended in the future, and the network layer expects a fresh packet object for its serialization pipeline.

## Data Pipeline

The Cloud class is a critical link in the chain that transforms a static asset file into visible, moving clouds on the client's screen.

> Flow:
> Weather Profile JSON File -> Server AssetLoader -> **Cloud.CODEC** -> **Cloud Instance** (in memory) -> Weather System Logic -> `toPacket()` -> Network Serialization Layer -> Client Rendering Engine

