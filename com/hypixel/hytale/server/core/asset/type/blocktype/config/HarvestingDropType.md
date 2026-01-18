---
description: Architectural reference for HarvestingDropType
---

# HarvestingDropType

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Configuration Model / Data Transfer Object

## Definition
```java
// Signature
public class HarvestingDropType implements NetworkSerializable<Harvesting> {
```

## Architecture & Concepts
The HarvestingDropType class is a passive data structure that defines the rules for harvesting a specific block. It is not an active service or manager; instead, it serves as a data contract, deserialized from asset configuration files at server startup.

Its primary role is to encapsulate two key pieces of information: the specific item required to harvest a block (if any) and the list of items that should be dropped upon a successful harvest.

This class forms a critical link between the server's static asset configuration and the dynamic game state. By implementing the NetworkSerializable interface, it provides a standardized way to convert its configuration data into a `Harvesting` network packet, which is then synchronized with game clients. This ensures that client-side predictions and server-side logic operate on the same harvesting rules.

The static CODEC field is the cornerstone of this class's integration into the engine's asset pipeline. The BuilderCodec defines the schema for parsing this object from a file and includes late-bound validators to ensure data integrity across the entire asset ecosystem, for example, by verifying that the specified Item and ItemDropList actually exist.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale asset loading system via the static `CODEC` field. During server bootstrap, the asset manager scans for block configuration files and uses this codec to deserialize the harvesting rules into HarvestingDropType objects.
- **Scope:** An instance of HarvestingDropType has its lifetime tied to the parent asset that contains it, typically a `BlockType` configuration. It is loaded into memory at server startup and persists for the entire server session.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection when the server's asset registry is cleared, which normally occurs during a full server shutdown.

## Internal State & Concurrency
- **State:** The internal state consists of `itemId` and `dropListId`. While the fields are not declared as final, the object should be considered **effectively immutable** after the asset loading phase is complete. Modifying its state at runtime is a severe anti-pattern that can lead to game state desynchronization.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be instantiated and populated on a single thread during the asset loading process. After initialization, it can be safely *read* by multiple threads (e.g., world simulation threads) because its state is not expected to change. Any runtime modification would introduce a race condition.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | Harvesting | O(1) | Serializes the object's state into a network-ready `Harvesting` packet for client synchronization. |
| getItemId() | String | O(1) | Returns the asset identifier for the item required to perform the harvest. |
| getDropListId() | String | O(1) | Returns the asset identifier for the `ItemDropList` used to generate drops. |
| withoutDrops() | HarvestingDropType | O(1) | Returns null. This method is likely used within a compositional or inheritance pattern to represent a state where no drops are possible. |

## Integration Patterns

### Standard Usage
This object is not intended to be used directly. Instead, game logic retrieves it from a parent configuration object, such as a `BlockType`, to determine the outcome of a player action.

```java
// This object is typically accessed via a parent configuration asset.
// Assume 'block' is a fully loaded BlockType asset instance.
HarvestingDropType harvestInfo = block.getHarvestingInfo();

if (harvestInfo != null) {
    String requiredToolId = harvestInfo.getItemId();
    String dropListId = harvestInfo.getDropListId();

    // Game logic would then use these IDs to check player inventory
    // and query the ItemDropList service to generate items.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new HarvestingDropType()`. Doing so bypasses the asset system's deserialization and validation pipeline, leading to unvalidated and disconnected data that the rest of the engine cannot recognize.
- **Runtime Modification:** Do not modify the `itemId` or `dropListId` fields after the server has started. This will break the data contract established at load time and cause unpredictable behavior and desynchronization between the server and clients.

## Data Pipeline
The primary flow for this data is from a static configuration file on disk, through the server's in-memory asset system, and finally into a network packet destined for the game client.

> Flow:
> BlockType Asset File -> AssetManager with CODEC -> **HarvestingDropType** (In-Memory) -> `toPacket()` -> `Harvesting` Packet -> Network Layer -> Game Client

