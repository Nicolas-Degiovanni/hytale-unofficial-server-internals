---
description: Architectural reference for Edge
---

# Edge

**Package:** com.hypixel.hytale.server.core.asset.type.trail.config
**Type:** Transient

## Definition
```java
// Signature
public class Edge {
```

## Architecture & Concepts
The Edge class is a server-side configuration data model. It represents the visual properties, specifically width and color, of a segment within a trail particle effect asset.

Its primary role is to act as an intermediate representation between a persistent asset definition (e.g., a JSON file on disk) and a network-transmissible protocol message. The class itself does not contain any logic; it is a pure data holder. Deserialization is managed externally via the static CODEC field, which allows for complex configuration patterns like property inheritance from a parent definition. This design decouples the asset file format from the network protocol, allowing either to change independently.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale codec system during asset deserialization. The static `CODEC` field is invoked by a higher-level asset loader which parses the trail configuration files. Direct instantiation is an anti-pattern.
- **Scope:** The lifetime of an Edge object is bound to its parent trail asset. It persists in memory as long as the corresponding trail asset is loaded on the server.
- **Destruction:** The object is marked for garbage collection when the server unloads the parent asset, for example, during a zone change or server shutdown.

## Internal State & Concurrency
- **State:** The state of an Edge object is **Mutable**. Its fields, `width` and `color`, are populated by the `BuilderCodec` after the constructor is called. The state is intended to be written once during asset loading and treated as immutable thereafter.
- **Thread Safety:** This class is **Not Thread-Safe**. It contains no internal locking or synchronization. It is designed to be populated on a single asset-loading thread and subsequently read by the main game thread. Unsynchronized, concurrent access will lead to undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | static BuilderCodec<Edge> | N/A | The public contract for serializing and deserializing an Edge instance from a keyed data source. |
| toPacket() | com.hypixel.hytale.protocol.Edge | O(1) | Transforms this configuration object into its corresponding network protocol packet for client transmission. |

## Integration Patterns

### Standard Usage
The class is not intended for direct use in typical game logic. Its primary interaction is converting the loaded configuration into a network packet when a trail effect needs to be spawned and communicated to clients.

```java
// Assume 'trailAsset' is a loaded configuration containing an Edge
Edge edgeConfig = trailAsset.getEdgeDefinition();

// Convert the configuration to a network-ready packet
com.hypixel.hytale.protocol.Edge edgePacket = edgeConfig.toPacket();

// Add the packet to an outgoing message
worldEntity.sendEffectPacket(edgePacket);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new Edge()`. The object will be in an uninitialized state with a width of 0.0 and a default, invalid color. All creation must be handled by the asset system via the `CODEC`.
- **Runtime Modification:** Do not modify the state of an Edge object after it has been loaded. This can lead to inconsistent visual effects for different players and is not thread-safe.

## Data Pipeline
The Edge class serves as a bridge, translating static configuration data into a dynamic network message.

> Flow:
> Trail Asset File (JSON) -> Asset Deserializer (using `Edge.CODEC`) -> **Edge instance** -> `toPacket()` call -> `protocol.Edge` Packet -> Network Layer -> Game Client

