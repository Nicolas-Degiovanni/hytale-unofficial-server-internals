---
description: Architectural reference for ModelOverride
---

# ModelOverride

**Package:** com.hypixel.hytale.server.core.asset.type.entityeffect.config
**Type:** Configuration Model

## Definition
```java
// Signature
public class ModelOverride implements NetworkSerializable<com.hypixel.hytale.protocol.ModelOverride> {
```

## Architecture & Concepts
The ModelOverride class is a data-centric configuration object that defines a set of visual overrides for an entity. It serves as a critical bridge between the server-side asset configuration system and the client-side rendering engine. Its primary purpose is to specify an alternative 3D model, texture, or set of animations for an entity, typically as part of an entity effect.

The design is centered around the static **CODEC** field, which integrates deeply with the engine's reflection-based asset loading pipeline. This codec is responsible for deserializing a ModelOverride definition from a data file (e.g., JSON) into a memory-resident Java object.

By implementing the NetworkSerializable interface, this class also defines a contract for its conversion into a network packet. This dual role is a key architectural pattern, eliminating the need for separate configuration and protocol-level data structures and ensuring that data sent to the client is a direct representation of the server's configuration.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the engine's asset loading system via the static **CODEC** field. The protected constructor strictly prohibits direct instantiation, ensuring that all objects are correctly initialized from a valid asset source.
- **Scope:** The lifetime of a ModelOverride object is bound to the parent asset that contains it, such as an EntityEffect configuration. It is loaded into memory when the parent asset is requested and persists as long as that asset is retained by the AssetManager.
- **Destruction:** The object is eligible for garbage collection when its parent asset is unloaded from memory. There are no explicit cleanup or disposal methods.

## Internal State & Concurrency
- **State:** The object's state is mutable during the asset deserialization process, where the **CODEC** populates its fields. After this initial loading phase, it is intended to be treated as an immutable data container. Its state consists of optional string references to model and texture assets, and a map of animation sets.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, populated, and managed on the main server thread or a dedicated asset loading thread. Concurrent modification after initialization will lead to undefined behavior and state corruption. Read-only access from multiple threads after the initial load is safe, but not a standard use case.

## API Surface
The public API is minimal, reflecting its role as a data holder.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.ModelOverride | O(N) | Converts the server-side configuration object into its network protocol equivalent. N is the number of animation sets. This is the primary mechanism for synchronizing visual overrides with the client. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in typical game logic. It is an internal component of the asset and entity effect systems. A higher-level system would retrieve a configured ModelOverride and use it to generate a network packet.

```java
// Example of an entity effect system preparing to send an update
// Note: This is a conceptual example. You would not typically interact with ModelOverride directly.

EntityEffect effect = assetManager.get("hytale:fire_aura");
ModelOverride override = effect.getModelOverride();

if (override != null) {
    // Convert the override to a packet to be sent to the client
    com.hypixel.hytale.protocol.ModelOverride packetData = override.toPacket();
    
    // Attach packetData to a larger entity update packet
    entityUpdatePacket.setModelOverride(packetData);
    networkManager.send(player, entityUpdatePacket);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ModelOverride()`. The protected constructor exists to prevent this. Bypassing the asset system's **CODEC** will result in an uninitialized and useless object.
- **Post-Load Mutation:** Do not modify the fields of a ModelOverride instance after it has been loaded. This breaks the "configuration-as-code" paradigm and can cause severe desynchronization between the server's state and what clients perceive.
- **State Caching:** Do not cache the result of `toPacket()`. The conversion is lightweight, and caching can lead to stale data if asset hot-reloading is ever implemented.

## Data Pipeline
ModelOverride acts as a transformation stage in the data flow from disk configuration to client-side rendering.

> Flow:
> Asset File (JSON) -> Engine Deserializer (using **CODEC**) -> **ModelOverride** Instance -> `toPacket()` call -> `com.hypixel.hytale.protocol.ModelOverride` Packet -> Network Layer -> Client Render Engine

