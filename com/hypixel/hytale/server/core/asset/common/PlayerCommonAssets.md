---
description: Architectural reference for PlayerCommonAssets
---

# PlayerCommonAssets

**Package:** com.hypixel.hytale.server.core.asset.common
**Type:** Transient

## Definition
```java
// Signature
public class PlayerCommonAssets {
```

## Architecture & Concepts
The PlayerCommonAssets class is a stateful, short-lived object that manages the asset synchronization state for a single connecting player. Its primary function is to calculate the "asset delta"—the precise set of assets a client is missing and must download from the server.

This class acts as a server-side calculator during the client connection handshake. It is initialized with a manifest of all assets required by the server. It then processes a report from the client detailing which of those assets the client already has cached. The result of this reconciliation is an internal list of assets that the server must then stream to the client.

This component is critical for minimizing network bandwidth and reducing client loading times by ensuring only necessary assets are ever transmitted.

### Lifecycle & Ownership
- **Creation:** An instance is created by a connection-handling service for each player when they enter the asset negotiation phase of the connection sequence. It is seeded with the complete list of assets the server currently requires.
- **Scope:** The object's lifetime is extremely brief, scoped exclusively to a single asset negotiation transaction for one player. It does not persist beyond this phase.
- **Destruction:** Once the asset delta has been calculated by calling the `sent` method, the object has fulfilled its purpose. It holds the final state but is typically discarded and becomes eligible for garbage collection immediately after the asset streaming service has been tasked with sending the delta.

## Internal State & Concurrency
- **State:** This class is highly mutable. Its internal maps, `assetMissing` and `assetSent`, are modified as part of its core reconciliation logic. It is designed as a single-use, transactional state machine and does not cache data across multiple operations.
- **Thread Safety:** **This class is not thread-safe.** It uses non-concurrent collections from the fastutil library and performs no internal locking. All method calls on a given instance must be serialized and confined to a single thread, typically the player's dedicated network processing thread. Unsynchronized access from multiple threads will result in data corruption and unpredictable behavior.

## API Surface
The public contract is minimal, consisting of the constructor and a single processing method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| sent(Asset[] hashes) | void | O(N+M) | Reconciles required assets against the client's inventory. Throws a RuntimeException if the client reports an asset the server did not request. |

**Warning:** The method name `sent` is potentially misleading. This method does not send assets; it *processes* a report of assets the client already possesses to calculate which ones *need to be sent*. Similarly, the internal `assetSent` map does not contain assets that have been sent, but rather the calculated delta of assets that *must* be sent.

## Integration Patterns

### Standard Usage
The class is used in a three-step process: instantiate with all required assets, process the client's inventory, and then use the resulting state to dispatch work to an asset streaming system.

```java
// 1. A service determines the full set of assets this server requires.
Asset[] requiredAssets = serverManifest.getAllAssets();

// 2. An instance is created for the connecting player.
PlayerCommonAssets assetCalculator = new PlayerCommonAssets(requiredAssets);

// 3. The client sends a packet with the hashes of assets it already has.
Asset[] clientHashes = networkPacket.getAssetHashes();

// 4. The calculator determines the delta.
// This mutates the internal state of assetCalculator.
assetCalculator.sent(clientHashes);

// 5. The asset delta is retrieved and passed to the streaming service.
// NOTE: The class provides no public getter; access to the internal `assetSent`
// map is assumed to be done via reflection or a package-private accessor.
// Map<String, String> assetsToSend = assetCalculator.assetSent;
// assetStreamingService.queueAssetsForPlayer(player, assetsToSend);
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not reuse a PlayerCommonAssets instance. It is a single-use calculator. Attempting to call `sent` multiple times or for different players will lead to incorrect asset calculations.
- **Concurrent Modification:** Do not access an instance from multiple threads. The internal collections are not thread-safe and will become corrupted.
- **Misinterpretation of State:** Do not assume the `assetSent` field contains assets that have already been transmitted. It contains the list of assets that still need to be sent to the client.

## Data Pipeline
This class sits between the server's static asset manifest and the dynamic asset streaming system, acting as the core logic for the asset negotiation phase.

> Flow:
> Server Asset Manifest -> **PlayerCommonAssets(constructor)** -> Client Asset Report (from Network) -> **sent(method)** -> Calculated Asset Delta (internal state) -> Asset Streaming Service -> Network Packet (to Client)

