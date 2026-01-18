---
description: Architectural reference for ParticleAttractor
---

# ParticleAttractor

**Package:** com.hypixel.hytale.server.core.asset.type.particle.config
**Type:** Configuration Model

## Definition
```java
// Signature
public class ParticleAttractor implements NetworkSerializable<com.hypixel.hytale.protocol.ParticleAttractor> {
```

## Architecture & Concepts

The ParticleAttractor is a pure data model that defines the physical properties of a force influencing particles within a particle effect. It is not an active system or service; rather, it serves as a configuration container that is deserialized from asset files and used to parameterize the server-side particle simulation engine.

Its primary architectural role is to act as a strongly-typed, in-memory representation of configuration data defined in external asset files (e.g., JSON). This is achieved through the static **CODEC** field, a `BuilderCodec` instance that declaratively maps asset file keys to the class's internal fields. This pattern decouples the data structure from the parsing logic, allowing for robust and maintainable asset loading.

The implementation of the `NetworkSerializable` interface is critical. It signifies that this server-side configuration has a direct counterpart in the network protocol. The class is responsible for translating itself into a network-ready packet object, which is then sent to clients to ensure visual effects are synchronized.

## Lifecycle & Ownership

-   **Creation:** A ParticleAttractor instance is never created directly using its constructor in standard game logic. It is instantiated exclusively by the engine's `Codec` system during the asset loading phase. The `BuilderCodec` uses the protected no-argument constructor and reflection to populate the object's fields from a data source.
-   **Scope:** The lifetime of a ParticleAttractor is bound to its parent particle effect asset. It persists in memory as a read-only configuration object for as long as the asset is loaded and referenced by the `AssetManager`.
-   **Destruction:** The object is marked for garbage collection when its parent particle effect asset is unloaded from memory, typically when a world is shut down or assets are reloaded.

## Internal State & Concurrency

-   **State:** The ParticleAttractor is effectively **immutable**. Although its fields are not marked as final, there are no public setters. State is populated once at creation time by the `Codec` system. This design ensures that configuration remains consistent and predictable throughout its lifecycle.
-   **Thread Safety:** This class is **thread-safe for read operations**. As an immutable data container, it can be safely accessed and shared across multiple threads (e.g., game logic thread, networking thread) without the need for locks or other synchronization primitives.

## API Surface

The public API is minimal, consisting primarily of data accessors and the network serialization contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get...() | various | O(1) | A suite of standard getters to access the physics properties. |
| toPacket() | com.hypixel.hytale.protocol.ParticleAttractor | O(1) | **Critical Method.** Serializes the object into its network protocol equivalent for transmission to the client. |

## Integration Patterns

### Standard Usage

A developer does not typically interact with this class directly. Instead, they define the attractor's properties within a particle effect's asset file. The engine consumes this data to configure simulations. Direct interaction is rare and limited to inspecting loaded asset data.

```java
// A developer defines the attractor in a data file, not in code.
// Example (conceptual JSON asset):
/*
{
  "format_version": "1.0",
  "particle_effect": {
    "attractor": {
      "Position": [0.0, 1.0, 0.0],
      "Radius": 10.0,
      "LinearAcceleration": [0.0, -9.8, 0.0]
    }
  }
}
*/

// The engine uses the static CODEC to load this data into a ParticleAttractor object.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not call `new ParticleAttractor(...)`. This creates a hardcoded, "magic" configuration that bypasses the asset pipeline. All particle physics should be defined in data files for modularity and designer iteration.
-   **State Mutation:** Do not use reflection to modify the fields of a ParticleAttractor after it has been loaded. The particle simulation engine and network layer assume this data is immutable. Unpredictable visual artifacts or server-client desynchronization will occur.

## Data Pipeline

The ParticleAttractor is a key component in the pipeline that transforms static asset definitions into synchronized, in-game visual effects.

> Flow:
> Particle Asset File (JSON) -> AssetManager with BuilderCodec -> **ParticleAttractor** (In-Memory Object) -> toPacket() -> Network Packet -> Client-Side Particle Renderer

