---
description: Architectural reference for RepulsionConfigPacketGenerator
---

# RepulsionConfigPacketGenerator

**Package:** com.hypixel.hytale.server.core.modules.entity.repulsion
**Type:** Utility

## Definition
```java
// Signature
public class RepulsionConfigPacketGenerator extends AssetPacketGenerator<String, RepulsionConfig, IndexedLookupTableAssetMap<String, RepulsionConfig>> {
```

## Architecture & Concepts
The RepulsionConfigPacketGenerator is a specialized component within the server's asset synchronization framework. Its sole responsibility is to translate server-side RepulsionConfig asset data into network-routable Packet objects. This class acts as a bridge between the server's internal asset representation and the client's, ensuring that clients have the correct entity repulsion data for physics calculations.

Architecturally, this class embodies the **Adapter Pattern**. It adapts the complex, server-native RepulsionConfig and IndexedLookupTableAssetMap objects into a flattened, serializable `UpdateRepulsionConfig` packet.

A key concept is the use of an IndexedLookupTableAssetMap. This map translates string-based asset identifiers (e.g., "player_heavy_repulsion") into compact integer indices. This is a critical network optimization that significantly reduces packet size by avoiding the transmission of redundant string data. The generator handles three distinct synchronization events:
1.  **Init:** A full synchronization, sent to a client upon connection to provide the complete set of repulsion configurations.
2.  **AddOrUpdate:** An incremental update, sent when existing configurations are modified or new ones are added during a live session.
3.  **Remove:** An instruction to a client to delete a specific configuration that is no longer valid.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the server's central asset management system during the bootstrap phase. It is typically registered as the designated packet generator for the RepulsionConfig asset type.
-   **Scope:** This is a long-lived object. Its lifecycle is tied to the server's main runtime, persisting for the entire session.
-   **Destruction:** De-referenced and garbage collected upon server shutdown when the parent asset management system is dismantled.

## Internal State & Concurrency
-   **State:** This class is **stateless**. It contains no member fields and all its methods operate exclusively on the parameters passed to them. Each method call is an independent, pure function.
-   **Thread Safety:** The stateless nature of this class makes it inherently **thread-safe**. It can be safely invoked from multiple threads without locks or synchronization, for example, if multiple asset updates were being processed in parallel.

## API Surface
The public contract is defined by its parent, AssetPacketGenerator, and is focused on creating packets for different synchronization scenarios.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Creates a full synchronization packet containing all specified assets. N is the number of assets. |
| generateUpdatePacket(assetMap, assets, query) | Packet | O(N) | Creates an incremental update packet for new or modified assets. N is the number of loaded assets. |
| generateRemovePacket(assetMap, removed, query) | Packet | O(N) | Creates a packet to instruct the client to remove specified assets. N is the number of removed assets. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay logic developers. It is an internal component of the asset pipeline, invoked automatically by the server's AssetService. When a RepulsionConfig asset is modified, the service identifies this generator as the handler for that asset type and calls the appropriate method to create a synchronization packet.

```java
// Pseudo-code from a hypothetical AssetService
// This logic is internal to the engine

AssetPacketGenerator generator = assetGenerators.get(RepulsionConfig.class);
Packet updatePacket = generator.generateUpdatePacket(assetMap, updatedAssets, query);
networkManager.broadcast(updatePacket);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new RepulsionConfigPacketGenerator()`. The asset system is responsible for creating and managing generator instances. Manual creation bypasses this system and will have no effect.
-   **Manual Packet Construction:** Never manually create an `UpdateRepulsionConfig` packet. The asset system guarantees that the integer indices in the packet are correctly synchronized with the client's lookup table. Manual creation will lead to asset desynchronization and client-side errors.

## Data Pipeline
This generator is a critical step in the server-to-client asset propagation pipeline. It transforms data from its persistent, high-level format into a transient, network-optimized format.

> Flow:
> Asset File (JSON/HOCON) -> Server AssetManager -> **RepulsionConfigPacketGenerator** -> UpdateRepulsionConfig Packet -> Network Layer -> Client AssetStore -> Client Physics Engine

