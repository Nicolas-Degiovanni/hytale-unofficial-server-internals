---
description: Architectural reference for SimpleAssetPacketGenerator
---

# SimpleAssetPacketGenerator

**Package:** com.hypixel.hytale.server.core.asset.packet
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class SimpleAssetPacketGenerator<K, T extends JsonAssetWithMap<K, M>, M extends AssetMap<K, T>> extends AssetPacketGenerator<K, T, M> {
```

## Architecture & Concepts

The SimpleAssetPacketGenerator is an abstract base class that provides a simplified contract for serializing asset state into network packets. It serves as a strategic specialization of the more complex AssetPacketGenerator, intentionally trading network efficiency for implementation simplicity.

Its core design principle is the deliberate omission of delta-based updates. While a standard AssetPacketGenerator might leverage an AssetUpdateQuery to generate highly optimized, minimal-change packets, this class ignores the query. Instead, it mandates that subclasses generate packets representing the *entire current state* of the asset collection upon any update.

This architecture is best suited for asset types that are small in number and size, where the overhead of calculating precise changes outweighs the benefit of a smaller network payload. It embodies the Template Method Pattern, defining a skeleton algorithm in its public methods and deferring the concrete packet creation steps to subclasses.

### Lifecycle & Ownership
- **Creation:** As an abstract class, SimpleAssetPacketGenerator is never instantiated directly. Concrete subclasses are instantiated by the server's core asset management system during the bootstrap phase. These generators are then registered against specific asset types.
- **Scope:** A registered generator instance is a stateless singleton that persists for the entire server session. Its methods are invoked by the asset system whenever a corresponding asset is created, modified, or deleted.
- **Destruction:** The object is dereferenced and eligible for garbage collection when the server shuts down and the asset management system is dismantled.

## Internal State & Concurrency
- **State:** This class is entirely stateless. It maintains no internal fields, caches, or mutable data structures. All required context is provided via method arguments.
- **Thread Safety:** The class itself is inherently thread-safe due to its stateless nature. However, the asset collections passed into its methods are **not** guaranteed to be thread-safe. The calling context, typically the main server thread, is responsible for ensuring that modifications to the asset maps do not occur concurrently with calls to the packet generation methods. External synchronization is required.

## API Surface

The public contract is defined by its parent, but its behavior is specialized. The primary interaction for implementers is with the protected abstract methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, loadedAssets) | Packet | O(N) | **Abstract.** Subclasses must implement this to create a packet containing the full initial state of all specified assets. N is the number of assets. |
| generateUpdatePacket(assetMap, loadedAssets, query) | Packet | O(N) | Overrides the parent to ignore the AssetUpdateQuery. This method delegates to the simpler protected generateUpdatePacket, effectively forcing a full-state resync on any update. |
| generateRemovePacket(assetMap, removed, query) | Packet | O(N) | Overrides the parent to ignore the AssetUpdateQuery. Delegates to the simpler protected generateRemovePacket. May return null if no packet is necessary for a removal operation. |

## Integration Patterns

### Standard Usage

A developer does not call methods on this class directly. Instead, they extend it to provide a simple packet generation strategy for a custom asset type.

```java
// A concrete implementation for a specific asset type, e.g., server-wide announcements.
public class AnnouncementPacketGenerator extends SimpleAssetPacketGenerator<String, AnnouncementAsset, AnnouncementAssetMap> {

    @Override
    public Packet generateInitPacket(AnnouncementAssetMap map, Map<String, AnnouncementAsset> assets) {
        // Creates a packet containing all initial announcements.
        return new PacketAnnouncementsInit(assets.values());
    }

    @Override
    protected Packet generateUpdatePacket(AnnouncementAssetMap map, Map<String, AnnouncementAsset> assets) {
        // On any change, re-sends all current announcements.
        return new PacketAnnouncementsUpdate(assets.values());
    }

    @Override
    @Nullable
    protected Packet generateRemovePacket(AnnouncementAssetMap map, Set<String> removedKeys) {
        // Creates a packet telling the client which announcement keys to remove.
        return new PacketAnnouncementsRemove(removedKeys);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Misuse for Large Datasets:** **CRITICAL:** Do not use this class for asset types with a large or unbounded number of instances, such as world entities or player inventories. The "full state" update approach is fundamentally unscalable and will cause severe network latency and bandwidth consumption. For such cases, extend the base AssetPacketGenerator and implement proper delta-handling using the AssetUpdateQuery.
- **Complex Logic in Overrides:** The purpose of this class is to enable a trivial implementation. If your `generateUpdatePacket` method requires complex logic to calculate differences, you are fighting the framework's design. You should not be using this class and should instead implement the more powerful AssetPacketGenerator directly.

## Data Pipeline

This component acts as a serializer in the server's asset synchronization pipeline. It is the final step before an asset state change is handed off to the networking layer.

> Flow:
> Server-side Asset Change -> Asset Management Service -> **SimpleAssetPacketGenerator (Implementation)** -> Network Packet -> Protocol Serialization Layer -> Client
---

