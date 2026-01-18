---
description: Architectural reference for ViewBobbingPacketGenerator
---

# ViewBobbingPacketGenerator

**Package:** com.hypixel.hytale.builtin.adventure.camera.asset.viewbobbing
**Type:** Utility

## Definition
```java
// Signature
public class ViewBobbingPacketGenerator extends SimpleAssetPacketGenerator<MovementType, ViewBobbing, AssetMap<MovementType, ViewBobbing>> {
```

## Architecture & Concepts
The ViewBobbingPacketGenerator is a specialized component within the server's asset synchronization framework. It acts as a serialization bridge, translating server-side changes in ViewBobbing asset data into network-ready packets for clients.

Its primary responsibility is to handle the lifecycle of ViewBobbing assets—which define camera motion effects for different player movement types—and encode their state into UpdateViewBobbing packets. By extending SimpleAssetPacketGenerator, it integrates directly into the engine's core asset pipeline, allowing the system to automatically generate the correct network updates when game designers modify ViewBobbing configuration files.

This class decouples the in-memory representation of an asset (AssetMap of MovementType to ViewBobbing) from its network protocol representation (UpdateViewBobbing). This separation is critical for maintaining a clean boundary between the asset management system and the low-level network layer.

### Lifecycle & Ownership
- **Creation:** An instance of ViewBobbingPacketGenerator is created and registered by the asset system during server initialization. It is typically associated with the asset type definition for ViewBobbing profiles.
- **Scope:** The instance is a stateless singleton, persisting for the entire lifetime of the server process. It is designed to be reused for all ViewBobbing asset updates.
- **Destruction:** The object is garbage collected upon server shutdown when the asset system is dismantled.

## Internal State & Concurrency
- **State:** This class is completely stateless. It contains no instance fields and all operations are performed on data passed in via method arguments. Its behavior is deterministic based on its inputs.
- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. Its methods can be invoked concurrently from any thread without requiring external synchronization. The caller is responsible for ensuring the thread safety of the asset maps passed into the methods.

## API Surface
The public contract is defined by its parent, SimpleAssetPacketGenerator. These methods are invoked by the asset management system, not by general game logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(map, assets) | Packet | O(N) | Generates a packet to fully synchronize all existing ViewBobbing profiles to a client. N is the total number of profiles. |
| generateUpdatePacket(map, assets) | Packet | O(N) | Generates a packet for new or modified ViewBobbing profiles. N is the number of changed profiles. |
| generateRemovePacket(map, removed) | Packet | O(N) | Generates a packet to signal the removal of specific ViewBobbing profiles. N is the number of removed profiles. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by feature developers. It is an internal component of the asset pipeline. The engine's AssetManager invokes its methods in response to file system changes or dynamic asset modifications.

A conceptual view of its invocation by the framework:

```java
// Pseudo-code showing framework-level usage
AssetType<MovementType, ViewBobbing> viewBobbingType;
ViewBobbingPacketGenerator generator = viewBobbingType.getPacketGenerator();

// On initial sync for a player
Map<MovementType, ViewBobbing> allProfiles = assetStore.getAll(viewBobbingType);
Packet initPacket = generator.generateInitPacket(assetStore, allProfiles);
player.send(initPacket);

// On a hot-reload of an asset
Map<MovementType, ViewBobbing> updatedProfiles = getUpdatedAssets();
Packet updatePacket = generator.generateUpdatePacket(assetStore, updatedProfiles);
server.broadcast(updatePacket);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not create instances using `new ViewBobbingPacketGenerator()`. The asset system is responsible for managing the lifecycle of this object. Manually creating it bypasses its registration with the asset pipeline.
- **Manual Invocation:** Avoid calling the generate methods directly to send game state. This class is strictly for asset synchronization. Using it for other purposes breaks the architectural separation of concerns.

## Data Pipeline
This generator functions as a key step in the server-to-client asset data flow. It serializes configuration data for transport over the network.

> Flow:
> Server Asset Change (e.g., JSON file modified) -> AssetManager detects change -> **ViewBobbingPacketGenerator** is invoked -> UpdateViewBobbing packet is created and cached -> Packet sent to Network Layer -> Client receives packet -> Client-side AssetStore updates ViewBobbing profiles.

