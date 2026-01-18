---
description: Architectural reference for ParticleSpawnerPacketGenerator
---

# ParticleSpawnerPacketGenerator

**Package:** com.hypixel.hytale.server.core.asset.type.particle
**Type:** Transient

## Definition
```java
// Signature
public class ParticleSpawnerPacketGenerator extends DefaultAssetPacketGenerator<String, ParticleSpawner> {
```

## Architecture & Concepts
The ParticleSpawnerPacketGenerator is a specialized, stateless transformer component within the server's Asset Synchronization Subsystem. Its sole responsibility is to act as a serialization bridge, converting the server's in-memory representation of ParticleSpawner assets into the concrete `UpdateParticleSpawners` network packet.

This class embodies the **Strategy Pattern**. It is one of many possible implementations of the `DefaultAssetPacketGenerator` contract, each tailored to a specific asset type. This design decouples the high-level asset management and synchronization orchestrator from the low-level details of network packet construction for particle effects. The orchestrator can operate on the generic `DefaultAssetPacketGenerator` interface, delegating the asset-specific serialization logic to this class at runtime.

This component is critical for synchronizing dynamic game assets, such as those modified by developers during live server operation (hot-reloading) or loaded for the first time.

### Lifecycle & Ownership
- **Creation:** This class is not meant to be managed directly. It is instantiated by a higher-level service, typically an `AssetPacketGeneratorRegistry` or a dependency injection framework, during server bootstrap. It is registered against the ParticleSpawner asset type.
- **Scope:** As a stateless object, an instance can be treated as a singleton and shared throughout the server's lifetime. However, it is often handled as a transient dependency, created on-demand by the asset synchronization pipeline and immediately discarded.
- **Destruction:** The object holds no resources or state, making its destruction trivial. It is garbage collected when no longer referenced by the registry or the active synchronization process.

## Internal State & Concurrency
- **State:** **Stateless**. This class contains no member fields and maintains no state between invocations. All data required for its operations is provided via method arguments. The output of any method is determined solely by its inputs.
- **Thread Safety:** **Inherently Thread-Safe**. Due to its stateless nature, a single instance of ParticleSpawnerPacketGenerator can be safely invoked by multiple threads concurrently without risk of race conditions or data corruption.
    - **Warning:** While the generator itself is thread-safe, the collections passed as arguments (e.g., `assets`, `loadedAssets`) are not. The calling system is responsible for ensuring that these collections are not mutated by another thread during a packet generation operation.

## API Surface
The public API is defined by its parent, `DefaultAssetPacketGenerator`, and provides methods to handle the three fundamental asset synchronization states: initialization, incremental update, and removal.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(map, assets) | Packet | O(N) | Constructs a full synchronization packet for all known ParticleSpawner assets. Used to initialize a client's state. |
| generateUpdatePacket(loadedAssets) | Packet | O(N) | Constructs a delta packet containing only new or modified ParticleSpawner assets. |
| generateRemovePacket(removed) | Packet | O(N) | Constructs a delta packet containing the string keys of assets that have been unloaded from the server. |

## Integration Patterns

### Standard Usage
This class should never be invoked directly. It is designed to be resolved and used by the asset synchronization service, which manages the lifecycle of asset updates and dispatches work to the appropriate registered generator.

```java
// Hypothetical usage within an AssetSynchronizationService
// DO NOT replicate this pattern; rely on the existing service.

// 1. A change in ParticleSpawner assets is detected.
Map<String, ParticleSpawner> updatedSpawners = getUpdatedParticleSpawners();

// 2. The service resolves the correct generator from a central registry.
DefaultAssetPacketGenerator generator = generatorRegistry.get(ParticleSpawner.class);

// 3. The generator is invoked to create the network packet.
Packet packet = generator.generateUpdatePacket(updatedSpawners);

// 4. The packet is dispatched to the network layer.
networkManager.sendToAllClients(packet);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ParticleSpawnerPacketGenerator()`. The asset system relies on a central registry to map asset types to packet generators. Bypassing this registry will prevent your assets from being synchronized correctly and breaks the core architectural pattern.
- **Stateful Extension:** Do not extend this class to add state. The design contract assumes generators are stateless and fully re-entrant. Adding state will introduce severe concurrency bugs.
- **Incorrect Packet Generation:** Do not call `generateInitPacket` for a small update. Clients rely on the `UpdateType` field within the packet to correctly apply the changes. Sending an `Init` packet will cause the client to discard its entire existing state and rebuild it, which is inefficient and may cause visual artifacts.

## Data Pipeline
The ParticleSpawnerPacketGenerator is a single, critical step in the server-to-client asset propagation pipeline.

> Flow:
> Asset File Change -> AssetManager Hot-Reload -> AssetSynchronizationService -> **ParticleSpawnerPacketGenerator** -> UpdateParticleSpawners Packet -> Network Layer -> Client AssetStore

