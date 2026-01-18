---
description: Architectural reference for BlockPlacementSettings
---

# BlockPlacementSettings

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Configuration Model

## Definition
```java
// Signature
public class BlockPlacementSettings implements NetworkSerializable<com.hypixel.hytale.protocol.BlockPlacementSettings> {
```

## Architecture & Concepts
BlockPlacementSettings is a server-side data model that encapsulates the rules and behaviors governing how a specific block type can be placed in the world. It is not a live game-world entity but rather a static, descriptive configuration object loaded from a block's core asset definition file.

The primary architectural role of this class is to serve as a bridge between the static asset configuration system and the runtime network protocol. It holds server-authoritative placement rules which are then translated into a network-optimized format for transmission to the client. This ensures that clients have the necessary information to provide placement previews and validation without needing access to the full server-side asset files.

The static `CODEC` field is the cornerstone of this system. It leverages the engine's `BuilderCodec` to declaratively map keys from an asset file (e.g., a JSON object) to the fields of this class. This provides a robust, type-safe mechanism for deserializing block configuration at server startup.

## Lifecycle & Ownership
- **Creation:** Instances are exclusively instantiated by the Hytale asset loading system via the static `CODEC`. This process occurs when the server deserializes `BlockType` assets from disk or a content pack during the bootstrap sequence. It is never created dynamically during gameplay.
- **Scope:** An instance of BlockPlacementSettings is owned by its parent `BlockType` asset. Its lifetime is bound to the lifetime of that asset. It persists in memory for the entire server session.
- **Destruction:** The object is marked for garbage collection only when its parent `BlockType` is unloaded, an event that typically occurs only during a full server shutdown or a comprehensive asset reload.

## Internal State & Concurrency
- **State:** This object is effectively **immutable** post-initialization. Its fields are populated once by the `CODEC` during asset loading and are not intended to be modified thereafter. It functions as a read-only container for configuration data.
- **Thread Safety:** The class is **thread-safe for read operations**. As its state is fixed after creation, multiple game logic threads can safely access its properties without requiring locks or other synchronization primitives. The `toPacket` method performs lookups but does not mutate the object's state.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.BlockPlacementSettings | Lookup | **Critical.** Translates this server-side model into its network protocol equivalent. This involves resolving string-based block IDs into integer indices for network efficiency. Throws IllegalArgumentException if an override block ID is not found in the asset registry. |
| getWallPlacementOverrideBlockId() | String | O(1) | Returns the asset ID of the block to use when this block is placed against a wall. |
| getFloorPlacementOverrideBlockId() | String | O(1) | Returns the asset ID of the block to use when this block is placed on a floor. |
| getCeilingPlacementOverrideBlockId() | String | O(1) | Returns the asset ID of the block to use when this block is placed on a ceiling. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most game logic. Instead, systems query the parent `BlockType` asset, which in turn exposes these placement settings. The `toPacket` method is invoked internally by the server when synchronizing asset properties with a connecting client.

```java
// Example of an internal system preparing data for the client
BlockType someBlock = BlockType.getAssetMap().get("hytale:stone");
BlockPlacementSettings serverSettings = someBlock.getPlacementSettings();

// The server's network layer will invoke this to create the packet
com.hypixel.hytale.protocol.BlockPlacementSettings packetData = serverSettings.toPacket();
client.send(packetData);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new BlockPlacementSettings()`. The protected constructor enforces this. All instances must be created by the asset loader to ensure they are correctly populated and registered.
- **State Mutation:** Do not use reflection or other means to modify the fields of this class after it has been loaded. Doing so would violate its immutability contract and lead to unpredictable behavior and desynchronization between the server and clients.

## Data Pipeline
The flow of data from configuration file to network packet is a one-way transformation.

> Flow:
> Block Asset File (e.g., JSON) -> Asset Loader (using CODEC) -> **BlockPlacementSettings** (In-memory, server-side) -> `toPacket()` method -> Protocol Packet (Integer-based IDs) -> Network Layer -> Client Rendering & Prediction

