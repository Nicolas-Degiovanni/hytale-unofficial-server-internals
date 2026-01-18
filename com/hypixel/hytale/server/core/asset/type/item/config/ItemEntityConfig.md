---
description: Architectural reference for ItemEntityConfig
---

# ItemEntityConfig

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Configuration Data Object

## Definition
```java
// Signature
public class ItemEntityConfig implements NetworkSerializable<com.hypixel.hytale.protocol.ItemEntityConfig> {
```

## Architecture & Concepts

The ItemEntityConfig class is a server-side data container that defines the physical and visual properties of an item when it exists as a dropped entity within the game world. It is not a live entity component but rather a static template, deserialized from asset configuration files at server startup.

Its primary architectural role is to bridge the gap between data-driven design (item properties defined in JSON or similar files) and the server's runtime systems (physics, rendering, and networking). The static CODEC field is the core mechanism for this, using the engine's reflection and codec system to map configuration keys to class fields.

A critical design aspect is its implementation of the NetworkSerializable interface. The corresponding toPacket method intentionally serializes only a subset of the configuration—specifically, visual properties like particle effects. Physics, lifetime (ttl), and pickup radius are exclusively server-side concerns and are not transmitted to the client, minimizing network overhead.

## Lifecycle & Ownership

-   **Creation:** Instances are created by the server's AssetManager during the asset loading phase. The static BuilderCodec, CODEC, is invoked to parse an item's asset file and construct a corresponding ItemEntityConfig object. Manual instantiation is strongly discouraged.
-   **Scope:** An instance of ItemEntityConfig has a singleton scope *per item type*. For example, a single ItemEntityConfig object is loaded for "hytale:iron_sword" and is shared as an immutable template for all iron sword entities created in the world. It persists for the entire server session.
-   **Destruction:** The object is eligible for garbage collection only when its defining asset is unloaded, which typically occurs during a full server shutdown or a manual asset hot-reload.

## Internal State & Concurrency

-   **State:** The object is highly mutable during its creation by the BuilderCodec. After the asset loading phase is complete, it must be treated as **effectively immutable**. All fields are protected and populated via reflection, resulting in a plain old data structure.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be instantiated and populated within the single-threaded context of the server's asset loading pipeline. Any subsequent reads from multiple threads (e.g., world threads) are safe only if the object is not modified.

    **WARNING:** Modifying a loaded ItemEntityConfig instance at runtime is a severe anti-pattern. Such an action would introduce race conditions and globally affect the behavior of every new item entity of that type.

## API Surface

The public API is minimal, primarily exposing read-only accessors and the network serialization contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.ItemEntityConfig | O(1) | Creates a network packet containing only client-relevant visual data. |
| getPhysicsValues() | PhysicsValues | O(1) | Returns the server-side physics properties for the item entity. |
| getPickupRadius() | float | O(1) | Returns the server-side distance at which players can pick up the item. |
| getTtl() | Float | O(1) | Returns the server-side time-to-live in seconds before the item despawns. |

## Integration Patterns

### Standard Usage

Developers do not typically interact with this class in Java code. Instead, they define its properties declaratively in an item's asset file. The engine handles the loading and application of this configuration.

A conceptual asset file might look like this:

```json
// in my_item.json
{
  "itemEntity": {
    "Physics": {
      "gravity": 5.0,
      "drag": 0.5
    },
    "PickupRadius": 1.75,
    "Lifetime": 600.0,
    "ParticleSystemId": "MyItemSparkle",
    "ParticleColor": "#FFD700",
    "ShowItemParticles": true
  }
}
```

The server engine then uses this configuration when creating a new item entity in the world, applying the physics and other server-side properties.

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new ItemEntityConfig()`. The object's state is meant to be populated entirely by the asset loading system via its CODEC. Manually created instances will be incomplete and bypass the data-driven design.
-   **Runtime Modification:** Never modify a loaded configuration object retrieved from an asset. This is not thread-safe and will cause unpredictable global changes. If dynamic properties are needed, they should be managed on a per-entity component, not on the shared configuration template.

## Data Pipeline

The flow of data from configuration file to network packet is central to this class's function.

> **Asset Loading Flow:**
> Item Asset File (e.g., JSON) -> Server AssetManager -> **ItemEntityConfig.CODEC** -> In-Memory **ItemEntityConfig** Instance

> **Network Serialization Flow:**
> Server creates Item Entity -> Reads from **ItemEntityConfig** Instance -> `toPacket()` -> `protocol.ItemEntityConfig` Packet -> Client Network Layer -> Client-side Rendering<ctrl63>

