---
description: Architectural reference for ModelParticle
---

# ModelParticle

**Package:** com.hypixel.hytale.server.core.asset.type.model.config
**Type:** Transient

## Definition
```java
// Signature
public class ModelParticle implements NetworkSerializable<com.hypixel.hytale.protocol.ModelParticle> {
```

## Architecture & Concepts
The ModelParticle class is a server-side data structure that acts as a blueprint for particle effects attached to entity models. It is not a live particle instance itself, but rather a configuration object that defines the properties, attachment points, and behavior of a particle effect that can be spawned from a model.

Its primary architectural role is to bridge the gap between static data defined in asset files (likely JSON) and the dynamic game world. This is achieved through two core mechanisms:

1.  **Codec-based Deserialization:** The static `CODEC` field is central to its design. The engine's asset loading system uses this codec to parse raw data from model definition files and instantiate validated ModelParticle objects. This ensures that all particle configurations are well-formed and consistent before they are ever used by the game logic.

2.  **Network Serialization:** By implementing the NetworkSerializable interface, this class defines a contract for converting its configuration into a network-ready packet object via the `toPacket` method. This allows the server to efficiently instruct clients to render a particle effect based on the asset definition, without sending the entire asset file.

In essence, a ModelParticle is a template read from disk, held in memory, and used to generate network commands for the rendering engine.

## Lifecycle & Ownership
-   **Creation:** Instances are almost exclusively created by the Hytale `Codec` framework during the server's asset loading phase. The static `CODEC` is invoked to deserialize a particle definition from a model asset file. Manual creation via `clone()` is a secondary, valid pattern for runtime duplication.

-   **Scope:** The lifetime of a ModelParticle instance is tied to its containing asset. It persists in memory as long as the parent model asset is loaded. Cloned instances are ephemeral and typically scoped to a single game tick or event handler.

-   **Destruction:** Instances are managed by the Java Garbage Collector. They are eligible for cleanup once the parent asset is unloaded and no other systems hold a reference.

## Internal State & Concurrency
-   **State:** The class is **highly mutable**. Fields like `scale` and `positionOffset` can be modified after instantiation, most notably via the `scale` method. This design allows for runtime adjustments to a particle template before it is spawned, such as scaling an effect based on an entity's size.

-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization primitives. All operations on a ModelParticle instance, especially mutations, must be performed on the main server thread to prevent race conditions and inconsistent state.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.ModelParticle | O(1) | Converts the configuration into a network protocol object for client transmission. |
| clone() | ModelParticle | O(1) | Creates a deep copy of the instance. Critical for preventing shared state mutation. |
| scale(float scale) | ModelParticle | O(1) | Mutates the internal scale and position offset. Returns a self-reference for chaining. |

## Integration Patterns

### Standard Usage
The intended pattern is for game logic to retrieve a pre-loaded ModelParticle template from an entity's asset definition, clone it to create a unique instance, and then convert it to a packet for spawning. This isolates any runtime modifications from the original asset template.

```java
// Retrieve the template from a loaded asset
ModelParticle particleTemplate = modelAsset.getParticle("footstep_dust");

// Clone the template to prevent mutating the shared asset
ModelParticle instanceToSpawn = particleTemplate.clone();

// Optionally modify the instance for this specific event
instanceToSpawn.scale(entity.getSizeScale());

// Convert to a network packet and send to clients
world.sendPacketToClients(instanceToSpawn.toPacket());
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ModelParticle()`. The static `CODEC` contains critical validation logic (e.g., checking for nulls, ensuring scale is positive) that is bypassed by the constructor. Always rely on the asset system for initial creation.

-   **Mutating Shared Templates:** Never call mutating methods like `scale` directly on an instance retrieved from a shared asset cache. This will permanently alter the template for all subsequent uses, leading to unpredictable visual bugs. Always `clone()` first.

-   **Cross-Thread Access:** Do not share a ModelParticle instance across threads. If a worker thread needs to calculate particle properties, it should operate on primitive data and pass the results back to the main thread for final instantiation and spawning.

## Data Pipeline
The flow of data from asset definition to client-side rendering is linear and well-defined. The ModelParticle serves as the in-memory representation at the core of this pipeline.

> Flow:
> Model Asset File (JSON) -> Codec Deserializer -> **ModelParticle Instance** -> Game Logic (e.g., Animation Event) -> `toPacket()` -> Network Layer -> Client Renderer

