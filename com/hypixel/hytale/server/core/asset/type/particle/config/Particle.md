---
description: Architectural reference for Particle
---

# Particle

**Package:** com.hypixel.hytale.server.core.asset.type.particle.config
**Type:** Configuration Model

## Definition
```java
// Signature
public class Particle implements NetworkSerializable<com.hypixel.hytale.protocol.Particle> {
```

## Architecture & Concepts

The Particle class is a server-side data model that represents the complete static configuration for a single particle type. It is not a live, in-world particle entity, but rather the blueprint from which those entities are created. Its primary architectural role is to serve as a strongly-typed, in-memory representation of a particle asset defined in a configuration file.

The most critical component of this class is the static **CODEC** field. This `BuilderCodec` instance defines the schema for deserializing a particle definition from a data source, such as a JSON file. It enforces a strict contract, including data types, validation rules (e.g., non-null fields, value ranges), and even provides metadata for Hytale's internal editing tools. This codec-centric design ensures that any loaded Particle object is guaranteed to be valid and conform to engine requirements.

Furthermore, by implementing the `NetworkSerializable` interface, this class acts as a translation layer between the server's configuration system and the game's network protocol. The `toPacket` method transforms this high-level configuration object into a low-level, optimized `protocol.Particle` object, which is the data structure sent to the client for rendering.

## Lifecycle & Ownership

-   **Creation:** A Particle object is almost exclusively instantiated by the Hytale `Codec` system during the server's asset loading phase. The static `CODEC` field is invoked to parse and validate a corresponding asset file. Direct instantiation via its constructor is strongly discouraged and bypasses the crucial validation pipeline.
-   **Scope:** An instance of Particle exists for the lifetime of its containing asset. It is loaded into memory, typically by a higher-level asset manager, and persists as long as that asset is required by the server. It is effectively a read-only singleton for a given particle definition.
-   **Destruction:** The object is eligible for garbage collection when its parent asset is unloaded from the asset management system, for example, when a server shuts down or a game mode deinitializes.

## Internal State & Concurrency

-   **State:** The internal state of a Particle object is populated once during deserialization and should be treated as **immutable** thereafter. It holds no runtime state and does not cache any data. It is a pure representation of the source asset file.
-   **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization primitives. The design assumes a "write-once, read-many" pattern. It is safe to be read by multiple threads only after it has been fully constructed and published by the single-threaded asset loading system.

    **WARNING:** Any external mutation of this object's fields after it has been loaded is a severe anti-pattern and will cause unpredictable behavior and race conditions across the server.

## API Surface

The public API is dominated by simple getters for its configuration properties. The most significant method is `toPacket`, which is central to its function.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.Particle | O(N) | Translates the configuration model into its network protocol equivalent. N is the number of animation frames. This is the canonical method for preparing particle data to be sent to a client. |

## Integration Patterns

### Standard Usage

Developers should never create Particle instances directly. Instead, they are retrieved from an asset management system. The primary interaction is reading its properties to inform game logic or relying on the engine to automatically serialize it for clients.

```java
// A hypothetical AssetManager retrieves the pre-loaded particle configuration
Particle flameConfig = assetManager.get(Particle.class, "hytale:fire_particle");

// Read a property for server-side logic
Size particleFrame = flameConfig.getFrameSize();
System.out.println("Flame particle frame size: " + particleFrame);

// The engine will internally call toPacket() when a particle effect
// needs to be synchronized with the client. Manual invocation is rare.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new Particle(...)`. This bypasses the validation and schema enforcement provided by the `CODEC`, potentially creating invalid objects that will cause runtime exceptions or visual artifacts.
-   **Post-Load Mutation:** Do not modify the fields of a Particle object after it has been loaded. These objects are shared across the server and are expected to be constant. Modifying a loaded particle's texture or animation at runtime is not supported and will affect all subsequent uses of that particle type.

## Data Pipeline

The Particle class is a key stage in the pipeline that transforms a human-readable asset file into a binary network message for the game client.

> Flow:
> Particle Asset File (.json) -> Hytale Codec System -> **Particle** (In-Memory Object) -> toPacket() -> `protocol.Particle` (Packet DTO) -> Network Layer -> Client Renderer

