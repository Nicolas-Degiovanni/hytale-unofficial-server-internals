---
description: Architectural reference for TagPatternPacketGenerator
---

# TagPatternPacketGenerator

**Package:** com.hypixel.hytale.server.core.asset.type.tagpattern
**Type:** Utility

## Definition
```java
// Signature
public class TagPatternPacketGenerator extends SimpleAssetPacketGenerator<String, TagPattern, IndexedLookupTableAssetMap<String, TagPattern>> {
```

## Architecture & Concepts
The TagPatternPacketGenerator is a specialized, stateless component within the server's asset synchronization framework. Its primary responsibility is to act as a translator between the server-side representation of TagPattern assets and the network packets required to synchronize those assets with connected clients.

This class is a concrete implementation of the generic SimpleAssetPacketGenerator, tailored specifically for TagPattern assets. It plays a critical role in network efficiency by converting string-based asset identifiers into compact integer indices. This conversion is handled by the **IndexedLookupTableAssetMap**, which maintains a consistent mapping between an asset's string key and its integer ID. The generator uses this map to create UpdateTagPatterns packets that are significantly smaller than they would be if they contained full string keys, reducing bandwidth and processing overhead for both the server and clients.

It effectively decouples the asset management logic from the low-level details of network packet construction. The asset system can focus on loading, unloading, and managing assets, delegating the task of serialization for network transport to this generator.

### Lifecycle & Ownership
- **Creation:** An instance of TagPatternPacketGenerator is typically created and owned by a higher-level service responsible for managing the lifecycle of TagPattern assets, such as a hypothetical TagPatternAssetService. It is not a globally managed singleton.
- **Scope:** The generator's lifetime is bound to its owning asset service. It is instantiated when the service starts and persists for the entire server session.
- **Destruction:** The object is eligible for garbage collection when its owning asset service is shut down and de-referenced, usually as part of the server shutdown sequence.

## Internal State & Concurrency
- **State:** This class is completely stateless. It contains no instance fields and all of its methods operate exclusively on the arguments provided. Its behavior is deterministic and depends only on its inputs.
- **Thread Safety:** The TagPatternPacketGenerator is unconditionally thread-safe. Due to its stateless nature, a single instance can be safely shared and invoked by multiple threads concurrently without any risk of race conditions or data corruption. No external locking or synchronization is required when calling its methods.

## API Surface
The public API consists of three methods for generating distinct types of synchronization packets.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Creates a full synchronization packet for an initial state dump. Used when a client first connects. Throws IllegalArgumentException if an asset key is not in the lookup map. |
| generateUpdatePacket(assetMap, loadedAssets) | Packet | O(N) | Creates an incremental update packet for new or modified assets. Used for live updates. Throws IllegalArgumentException if an asset key is not in the lookup map. |
| generateRemovePacket(assetMap, removed) | Packet | O(N) | Creates a packet to instruct clients to remove specific assets. Used when assets are unloaded. Throws IllegalArgumentException if an asset key is not in the lookup map. |

## Integration Patterns

### Standard Usage
The generator should be used by a central asset management service that orchestrates asset state changes. When the service modifies its collection of TagPattern assets, it invokes the appropriate generator method to create a packet, which is then broadcast to clients.

```java
// Example within a hypothetical asset service
// The generator and map are long-lived members of the service
private final TagPatternPacketGenerator generator = new TagPatternPacketGenerator();
private final IndexedLookupTableAssetMap<String, TagPattern> assetMap = ...;

// Method called when new assets are loaded from a mod
public void onAssetsLoaded(Map<String, TagPattern> newAssets) {
    // Update the central asset map first
    this.assetMap.addAll(newAssets.keySet());

    // Generate a packet to synchronize the changes
    Packet updatePacket = generator.generateUpdatePacket(this.assetMap, newAssets);

    // Broadcast the packet to all relevant clients
    networkManager.broadcast(updatePacket);
}
```

### Anti-Patterns (Do NOT do this)
- **State Mismatch:** Never call a generator method with an IndexedLookupTableAssetMap that is out of sync with the provided asset data. The lookup map must be updated *before* calling the generator. Failure to do so will result in an IllegalArgumentException and a server-side crash.
- **Inefficient Updates:** Do not use generateInitPacket for small, incremental changes. The Init packet is designed for a full state transfer and is highly inefficient for minor updates. Always use generateUpdatePacket or generateRemovePacket for live changes after the initial connection.

## Data Pipeline
The TagPatternPacketGenerator sits in the middle of the server-to-client asset synchronization pipeline. It is the component that transforms an abstract "asset state change" into a concrete, transmittable network message.

> Flow:
> Server Event (e.g., Mod Load) -> Asset Service updates `IndexedLookupTableAssetMap` -> **TagPatternPacketGenerator** is invoked -> `UpdateTagPatterns` Packet is created -> Network Manager serializes and sends packet -> Client receives and processes packet

