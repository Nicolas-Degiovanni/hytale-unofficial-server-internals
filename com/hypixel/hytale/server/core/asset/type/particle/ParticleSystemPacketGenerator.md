---
description: Architectural reference for ParticleSystemPacketGenerator
---

# ParticleSystemPacketGenerator

**Package:** com.hypixel.hytale.server.core.asset.type.particle
**Type:** Transient Utility

## Definition
```java
// Signature
public class ParticleSystemPacketGenerator extends DefaultAssetPacketGenerator<String, ParticleSystem> {
```

## Architecture & Concepts
The ParticleSystemPacketGenerator is a specialized component within the server's asset synchronization framework. Its sole responsibility is to act as a translation layer, converting server-side changes to ParticleSystem assets into network-routable packets for client consumption.

This class embodies the Strategy Pattern. The generic asset management system defines *when* asset packets must be generated (e.g., on initial client join, on a hot-reload), while concrete implementations like this one define *how* to generate those packets for a specific asset type. It isolates the logic for serializing ParticleSystem data from the core asset tracking and networking machinery, promoting modularity and extensibility.

It exclusively handles the lifecycle of particle assets—initial bulk load, incremental additions or updates, and removals—by creating precisely structured UpdateParticleSystems packets.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's central AssetService during the bootstrap process. The AssetService maintains a registry of packet generators for various asset types, and this class is registered to handle the ParticleSystem type.
-   **Scope:** This object is stateless and its lifecycle is tied to its owning AssetService. It persists for the entire duration of the server session.
-   **Destruction:** Marked for garbage collection upon server shutdown when the parent AssetService is de-referenced. It holds no native resources or persistent state, so no explicit cleanup is required.

## Internal State & Concurrency
-   **State:** This class is **completely stateless**. It contains no member fields and all operations are performed on data passed in via method arguments. The output of a method call is determined solely by its input.
-   **Thread Safety:** As a stateless object, ParticleSystemPacketGenerator is inherently **thread-safe**. It can be safely invoked by multiple threads concurrently without the need for locks or synchronization primitives. This is a critical design feature for the high-throughput demands of the server's asset pipeline.

## API Surface
The public contract is defined by its parent, DefaultAssetPacketGenerator, and is focused on three distinct lifecycle events for assets.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Creates a bulk-load packet containing all specified ParticleSystem assets. Used to synchronize a new client. |
| generateUpdatePacket(loadedAssets) | Packet | O(N) | Creates a packet containing newly added or modified ParticleSystem assets. Used for hot-reloading or dynamic updates. |
| generateRemovePacket(removed) | Packet | O(N) | Creates a packet listing the identifiers of ParticleSystem assets that have been unloaded and should be removed from the client. |

## Integration Patterns

### Standard Usage
This class is an internal framework component and is not intended for direct use by gameplay logic. The server's AssetService invokes it automatically when changes to ParticleSystem assets are detected.

```java
// Conceptual example from within the AssetService
// AssetService detects changes and routes them to the correct generator

Map<String, ParticleSystem> newParticles = getNewParticleSystems();
DefaultAssetPacketGenerator generator = getGeneratorForType(ParticleSystem.class); // Returns an instance of ParticleSystemPacketGenerator

Packet updatePacket = generator.generateUpdatePacket(newParticles);
networkManager.broadcast(updatePacket);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new ParticleSystemPacketGenerator()`. The asset framework is responsible for its creation and lifecycle. Direct instantiation bypasses the framework's registration and management logic.
-   **Stateful Extension:** Do not extend this class to add state. Its statelessness is fundamental to its thread-safety and reusability within the server architecture.

## Data Pipeline
This generator sits at a critical juncture in the server-to-client asset synchronization pipeline, transforming in-memory objects into a wire-format protocol.

> Flow:
> Server AssetManager (detects file change) -> AssetService (dispatches event) -> **ParticleSystemPacketGenerator** (creates packet) -> Network Layer (sends packet) -> Client (receives and updates local assets)

