---
description: Architectural reference for ResourceTypePacketGenerator
---

# ResourceTypePacketGenerator

**Package:** com.hypixel.hytale.server.core.asset.type.item
**Type:** Transient

## Definition
```java
// Signature
public class ResourceTypePacketGenerator extends DefaultAssetPacketGenerator<String, ResourceType> {
```

## Architecture & Concepts
The ResourceTypePacketGenerator is a specialized component within the server's Asset Synchronization Subsystem. Its sole responsibility is to translate the server-side, in-memory representation of ResourceType assets into network-routable Packet objects. It functions as a dedicated serializer, bridging the core asset management system with the low-level network protocol.

This class implements the strategy pattern defined by its parent, DefaultAssetPacketGenerator. The broader asset system delegates the packet creation logic for ResourceType assets to this specific implementation.

A critical architectural constraint enforced by this generator is its "all-or-nothing" initialization protocol. The `generateInitPacket` method explicitly rejects partial asset maps, ensuring that a connecting client receives a complete and consistent view of all available resource types. This prevents state desynchronization and runtime errors related to missing asset definitions.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the server's core asset management service during bootstrap. It is programmatically registered and mapped to the ResourceType asset category. It is not intended for manual creation.
-   **Scope:** The lifecycle of a ResourceTypePacketGenerator instance is tied to the server's runtime. It persists as long as the server is running and needs to synchronize ResourceType assets with clients.
-   **Destruction:** The object is de-referenced and becomes eligible for garbage collection upon server shutdown or a full reload of the asset management subsystem.

## Internal State & Concurrency
-   **State:** This class is **stateless**. It maintains no internal fields or cached data. Its methods are pure functions that operate exclusively on the arguments provided, producing a new Packet object on each invocation.
-   **Thread Safety:** This class is inherently **thread-safe**. As a stateless object, a single instance can be safely invoked by multiple threads concurrently without locks or other synchronization primitives. Callers must ensure that the collections passed as arguments are not mutated by other threads during method execution.

## API Surface
The public contract is defined by the DefaultAssetPacketGenerator parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Creates a full synchronization packet containing all ResourceType assets. Throws UnsupportedOperationException if the provided asset map is a partial snapshot. |
| generateUpdatePacket(loadedAssets) | Packet | O(N) | Creates a delta packet to add or update a set of ResourceType assets on the client. |
| generateRemovePacket(removed) | Packet | O(N) | Creates a delta packet to remove a set of ResourceType assets on the client, identified by their string keys. |

## Integration Patterns

### Standard Usage
This class is not designed for direct use by game logic or plugin developers. It is invoked exclusively by the server's internal asset synchronization service in response to changes in the ResourceType asset store.

```java
// The following is a conceptual example of internal engine usage.
// DO NOT replicate this pattern in game code.

// 1. The Asset System retrieves the registered generator for ResourceType
DefaultAssetPacketGenerator<String, ResourceType> generator = assetSystem.getPacketGeneratorFor(ResourceType.class);

// 2. On a full sync (e.g., player join), an Init packet is created
Map<String, ResourceType> allResourceTypes = assetStore.getAll(ResourceType.class);
Packet initPacket = generator.generateInitPacket(assetStore.getAssetMap(), allResourceTypes);
server.getNetworkManager().sendPacket(playerConnection, initPacket);

// 3. On a hot-reload, an Update packet is created
Map<String, ResourceType> changedAssets = getNewlyLoadedAssets();
Packet updatePacket = generator.generateUpdatePacket(changedAssets);
server.getNetworkManager().broadcastPacket(updatePacket);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new ResourceTypePacketGenerator()`. The asset system is responsible for its lifecycle. Direct instantiation bypasses the registration process and will have no effect.
-   **Partial Initialization:** Do not call `generateInitPacket` with a subset of the server's total ResourceType assets. This violates the class's core design contract and will trigger a runtime exception. All initial synchronizations must be complete.

## Data Pipeline
This generator acts as a specific step in the server-to-client asset synchronization pipeline. It is responsible for the serialization stage for its designated asset type.

> Flow:
> Server Asset Store (Change detected in a ResourceType) → Asset Synchronization Service → **ResourceTypePacketGenerator** → UpdateResourceTypes Packet (Serialization) → Server Network Layer → Client Network Layer → Client Asset Store (Deserialization & Update)

