---
description: Architectural reference for DefaultAssetPacketGenerator
---

# DefaultAssetPacketGenerator

**Package:** com.hypixel.hytale.server.core.asset.packet
**Type:** Base Class / Template

## Definition
```java
// Signature
public abstract class DefaultAssetPacketGenerator<K, T extends JsonAssetWithMap<K, DefaultAssetMap<K, T>>>
   extends SimpleAssetPacketGenerator<K, T, DefaultAssetMap<K, T>> {
```

## Architecture & Concepts
The DefaultAssetPacketGenerator is an abstract base class that establishes a standardized contract for serializing server-side asset state changes into network packets for client synchronization. It serves as a critical bridge between the server's high-level Asset Management system and the low-level Protocol Layer.

This class embodies the **Template Method** design pattern. It defines the skeleton of the asset synchronization algorithm, delegating the specific details of packet creation (initialization, update, removal) to concrete subclasses. Its heavy use of generics allows it to be a reusable and type-safe solution for a wide variety of keyed assets, such as models, sounds, or configurations.

The primary architectural role of this class is to enforce a consistent data flow for asset replication. By handling different change types—initial bulk load, incremental updates, and removals—it ensures that clients receive a coherent and accurate view of the server's asset state. The two final wrapper methods, `generateUpdatePacket` and `generateRemovePacket`, simplify the public contract for subclasses by abstracting away the full `assetMap` argument, promoting a cleaner and more focused implementation.

### Lifecycle & Ownership
- **Creation:** Concrete implementations of DefaultAssetPacketGenerator are not managed as singletons. They are typically instantiated by a higher-level service, such as an AssetSynchronizationManager, during server initialization. An instance is created for each distinct asset type that requires network synchronization.
- **Scope:** An instance of a generator persists for the entire server session. Its lifecycle is directly tied to the lifecycle of the asset type it manages.
- **Destruction:** Instances are eligible for garbage collection upon server shutdown or when the associated asset management system is unloaded. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** This class is fundamentally **stateless**. It does not maintain any internal cache, fields, or state between method invocations. Its methods operate exclusively on the arguments provided, making them pure transformations of data.
- **Thread Safety:** The class itself is inherently thread-safe due to its stateless design. However, this guarantee does not extend to the collections passed as arguments.

    **WARNING:** The caller is solely responsible for ensuring that the Map and Set arguments are not mutated by another thread while a `generate` method is executing. Failure to provide this external synchronization will lead to `ConcurrentModificationException` or other unpredictable race conditions.

## API Surface
The public contract is defined by the three abstract methods that subclasses must implement.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(map, loaded) | Packet | O(N) | Creates a packet for a full, initial synchronization of all assets. N is the total number of assets. |
| generateUpdatePacket(updated) | Packet | O(M) | Creates a packet containing only newly added or modified assets. M is the number of updated assets. |
| generateRemovePacket(removed) | Packet | O(R) | Creates a packet to instruct the client to unload specific assets. R is the number of removed keys. May return null if no packet is necessary. |

## Integration Patterns

### Standard Usage
A developer must extend this class to create a generator for a specific asset type. The server's asset synchronization service then invokes this concrete implementation to create packets in response to asset lifecycle events.

```java
// 1. Create a concrete implementation for a specific asset type, e.g., GameModels.
public class ModelAssetPacketGenerator extends DefaultAssetPacketGenerator<ModelKey, ModelAsset> {
    @Override
    public Packet generateInitPacket(DefaultAssetMap<ModelKey, ModelAsset> all, Map<ModelKey, ModelAsset> loaded) {
        // Implementation to write all loaded models to a new packet buffer
    }

    @Override
    public Packet generateUpdatePacket(Map<ModelKey, ModelAsset> updated) {
        // Implementation to write only the updated models to a packet buffer
    }

    @Override
    @Nullable
    public Packet generateRemovePacket(Set<ModelKey> removed) {
        // Implementation to write the keys of removed models to a packet buffer
    }
}

// 2. The Asset Sync Service uses the generator instance.
ModelAssetPacketGenerator generator = new ModelAssetPacketGenerator();
Map<ModelKey, ModelAsset> changedModels = assetService.getChangedModels();
Packet updatePacket = generator.generateUpdatePacket(changedModels);
networkManager.sendToAllClients(updatePacket);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not add fields to a subclass to store state between calls. Each packet generation must be an independent, idempotent operation based solely on its inputs. Storing state breaks thread safety and creates complex, unpredictable behavior.
- **Ignoring Nullability:** The `generateRemovePacket` method is annotated as Nullable. Implementations must be prepared to return null, and callers must perform a null check. Assuming a non-null packet will result in `NullPointerException`.
- **Bypassing the Generator:** Do not manually construct asset packets elsewhere in the codebase. All asset synchronization logic must flow through a dedicated generator to ensure consistency and maintainability.

## Data Pipeline
This class is a key transformation step in the server-to-client asset replication pipeline. It converts in-memory object representations into a serialized network format.

> Flow:
> AssetManager Event (e.g., Asset Loaded/Unloaded) -> AssetSynchronizationService -> **Concrete DefaultAssetPacketGenerator** -> `Packet` Object -> Protocol Serialization Layer -> TCP/UDP Stream -> Client

