---
description: Architectural reference for BlockSelectorToolData
---

# BlockSelectorToolData

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Data Model / Configuration

## Definition
```java
// Signature
public class BlockSelectorToolData implements NetworkSerializable<com.hypixel.hytale.protocol.BlockSelectorToolData> {
```

## Architecture & Concepts
BlockSelectorToolData is a server-side data model that encapsulates configuration properties specific to "Block Selector" tools. It is not a service or a manager; it is a simple data container whose state is defined entirely by game asset files (e.g., JSON).

Its primary architectural role is to serve as the in-memory representation of static item data deserialized by the Hytale **Codec** system. The static `CODEC` field is the key to its function, defining the schema for how raw data from an asset file is mapped into a strongly-typed Java object.

This class also acts as a bridge between the server's internal game state and the client-facing network protocol. By implementing the NetworkSerializable interface, it provides a standardized contract for converting its configuration into a network packet, ensuring that client-side logic can be synchronized with server-defined tool behavior.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale `Codec` system during the server's asset loading phase. The static `BuilderCodec` reads data from an item's configuration file, invokes the protected constructor, and populates the fields. Manual instantiation is unsupported and architecturally incorrect.
- **Scope:** The lifetime of a BlockSelectorToolData instance is bound to the parent item asset it configures. It persists in memory as long as its corresponding item definition is loaded, which is typically for the entire duration of a server session.
- **Destruction:** The object is eligible for garbage collection when the server unloads its game assets, usually during a shutdown or a full asset reload. There are no explicit cleanup or `close` methods.

## Internal State & Concurrency
- **State:** The state is mutable upon creation but should be treated as **effectively immutable** post-initialization. Its sole purpose is to hold configuration data loaded at startup. Modifying its state during runtime is an anti-pattern that can lead to inconsistent game behavior.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be instantiated on a dedicated asset loading thread and subsequently read by the main server game loop. All access from the game loop is expected to be read-only. If multi-threaded access is required, synchronization must be handled externally.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.BlockSelectorToolData | O(1) | Converts this server-side configuration object into its corresponding network packet DTO for client synchronization. |
| getDurabilityLossOnUse() | double | O(1) | Retrieves the configured amount of durability lost when the tool is used. |

## Integration Patterns

### Standard Usage
This object is not retrieved directly. It is accessed as a component of a larger item configuration object. Game logic queries this data to determine tool behavior.

```java
// Hypothetical example of retrieving the data from a parent item asset
ItemAsset tool = assetManager.get("my_awesome_pickaxe");
BlockSelectorToolData toolData = tool.getComponent(BlockSelectorToolData.class);

if (toolData != null) {
    double loss = toolData.getDurabilityLossOnUse();
    player.getHeldItem().applyDurabilityDamage(loss);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BlockSelectorToolData()`. The constructor is protected to enforce that all instances are created and populated by the asset loading and codec system. Bypassing this will result in an uninitialized object.
- **Runtime State Mutation:** Do not modify fields like `durabilityLossOnUse` after the server has started. This data is considered static configuration. Changes should be made in the source asset files and loaded via a server restart.

## Data Pipeline
The flow of data begins with a raw asset file on disk and ends with its properties being used by game logic and synchronized to the client.

> Flow:
> Item JSON Asset File -> Hytale Codec System -> **BlockSelectorToolData** (In-Memory Instance) -> Game Logic (Read-only access) -> toPacket() -> Network Protocol Layer -> Client

