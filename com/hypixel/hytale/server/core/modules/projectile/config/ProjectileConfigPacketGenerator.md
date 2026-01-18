---
description: Architectural reference for ProjectileConfigPacketGenerator
---

# ProjectileConfigPacketGenerator

**Package:** com.hypixel.hytale.server.core.modules.projectile.config
**Type:** Transient Utility

## Definition
```java
// Signature
public class ProjectileConfigPacketGenerator extends DefaultAssetPacketGenerator<String, ProjectileConfig> {
```

## Architecture & Concepts
The ProjectileConfigPacketGenerator is a specialized component within the server's Asset Synchronization Framework. It serves as a **Serialization Bridge**, translating the server's internal representation of projectile assets (ProjectileConfig) into a network-transferable data structure (UpdateProjectileConfigs packet).

This class embodies the **Strategy Pattern**, where the generic AssetService delegates the specific task of packet creation to a concrete implementation. This decouples the core asset management logic from the network protocol, allowing the asset system to remain agnostic of how its assets are serialized for clients.

Its primary responsibility is to ensure that clients receive accurate and timely updates about the state of projectile configurations. It handles three distinct synchronization scenarios:
1.  **Initial State:** Transmitting the entire set of projectile configurations to a newly connecting client.
2.  **Incremental Update:** Transmitting new or modified configurations, typically during a hot-reload event.
3.  **Removal:** Notifying clients that specific configurations have been unloaded and should be removed.

## Lifecycle & Ownership
-   **Creation:** This class is not a singleton. An instance is created and registered with the server's central AssetService during the bootstrap phase for the ProjectileConfig asset type. It is instantiated by the framework, not by user code.
-   **Scope:** The object's lifetime is tightly coupled to the registration of the ProjectileConfig asset type within the AssetService. It persists as long as the server is configured to manage and synchronize these assets.
-   **Destruction:** The instance is eligible for garbage collection when the server shuts down or, in a dynamic environment, if the ProjectileConfig asset type is programmatically unregistered.

## Internal State & Concurrency
-   **State:** The ProjectileConfigPacketGenerator is **stateless**. It does not maintain any internal fields, caches, or references to data between method invocations. Its methods operate exclusively on the arguments provided.
-   **Thread Safety:** This class is inherently **thread-safe**. As it is stateless, multiple threads can invoke its methods without risk of interference. However, the caller, typically the AssetService, is responsible for ensuring that the collections passed as arguments (e.g., assetMap, loadedAssets) are not mutated by other threads during packet generation. Failure to provide this external synchronization will lead to undefined behavior and potential server instability.

## API Surface
The public API is defined by its parent, DefaultAssetPacketGenerator, and provides the contract for the AssetService to manage the synchronization lifecycle.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Creates a packet containing all known projectile configurations. This is for initial client synchronization. Throws IllegalStateException on duplicate keys. |
| generateUpdatePacket(loadedAssets) | Packet | O(M) | Creates a packet for new or changed configurations. Used for hot-reloading. Throws IllegalStateException on duplicate keys. |
| generateRemovePacket(removed) | Packet | O(K) | Creates a packet to instruct clients to remove specified configurations by their keys. |

*N = total number of projectile configs; M = number of updated configs; K = number of removed configs.*

## Integration Patterns

### Standard Usage
This class is an internal component of the asset framework and should not be invoked directly by game logic developers. The server's AssetService uses it as part of the asset synchronization pipeline.

The conceptual flow within the asset management system is as follows:

```java
// Conceptual example within the AssetService or a related manager.
// This code would NOT be written by a typical developer.

// On server startup or full asset load...
Map<String, ProjectileConfig> allConfigs = assetLoader.loadAll();
Packet initPacket = projectilePacketGenerator.generateInitPacket(assetMap, allConfigs);
networkSystem.broadcastToClients(initPacket);

// On asset hot-reload...
Map<String, ProjectileConfig> updatedConfigs = assetLoader.loadChanges();
Packet updatePacket = projectilePacketGenerator.generateUpdatePacket(updatedConfigs);
networkSystem.broadcastToClients(updatePacket);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new ProjectileConfigPacketGenerator()`. The asset framework is responsible for its creation and registration. Attempting to use a rogue instance will have no effect on the official asset synchronization pipeline.
-   **Manual Invocation:** Do not call the generate methods from game logic. Manually creating and sending asset packets circumvents the authority of the AssetService, almost certainly leading to client-server state desynchronization. All asset changes must flow through the designated service.
-   **Incorrect Packet Usage:** Sending an `AddOrUpdate` packet to a newly connected client is a critical error. The client will lack the baseline configuration set, causing runtime failures. The `Init` packet is mandatory for the first synchronization.

## Data Pipeline
The generator is a single step in a larger data flow that propagates server-side asset changes to connected clients.

> Flow:
> Server Event (e.g., File System Change) -> AssetService Watcher -> AssetLoader reads ProjectileConfig from disk -> AssetService updates its internal registry -> **ProjectileConfigPacketGenerator** is invoked -> UpdateProjectileConfigs Packet is created -> Network Layer sends Packet to Client -> Client AssetStore receives Packet and updates local state.

