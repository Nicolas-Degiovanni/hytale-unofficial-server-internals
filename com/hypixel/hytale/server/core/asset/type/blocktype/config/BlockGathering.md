---
description: Architectural reference for BlockGathering
---

# BlockGathering

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Configuration Data Object

## Definition
```java
// Signature
public class BlockGathering implements NetworkSerializable<com.hypixel.hytale.protocol.BlockGathering> {
```

## Architecture & Concepts

The **BlockGathering** class is a data-driven configuration object that defines the outcomes of player interactions with a block, such as breaking, harvesting, or physics-based destruction. It is not a service or a manager, but rather a passive data container that encapsulates the rules for what items a block drops under various conditions.

This class is a fundamental component of the server's asset system, specifically as a sub-component of a parent **BlockType** asset. Its entire state is designed to be deserialized from a JSON configuration file using Hytale's powerful **BuilderCodec** system. The static **CODEC** field is the primary entry point for its construction, defining a strict contract for how the JSON data maps to the object's fields.

A key architectural feature is the `afterDecode` hook within its **CODEC** definition. This hook performs a critical post-processing step, transforming the raw array of tool data (`toolDataRaw`) into an optimized `Object2ObjectOpenHashMap` (`toolData`). This transformation ensures that lookups for tool-specific drop behavior during gameplay are highly efficient, using the tool's type ID as a key.

Furthermore, its implementation of **NetworkSerializable** signifies its role in server-client state synchronization. The server serializes this configuration into a network packet to inform the client of the block's gathering properties, which can be used for predictive client-side effects.

### Lifecycle & Ownership

-   **Creation:** An instance of **BlockGathering** is never instantiated directly via its constructor. It is exclusively created and populated by the Hytale **Codec** framework during the server's asset loading phase. The framework invokes the static **CODEC** field to deserialize a corresponding section from a block's JSON asset file.
-   **Scope:** The lifecycle of a **BlockGathering** object is strictly bound to its parent **BlockType** asset. It remains in memory for as long as the block type is registered and active in the game world.
-   **Destruction:** The object is de-referenced and becomes eligible for garbage collection when its parent **BlockType** is unloaded. This typically occurs during a server shutdown or a dynamic asset reload.

## Internal State & Concurrency

-   **State:** The object is mutable *only* during the asset deserialization process. Once the `afterDecode` hook in its **CODEC** has executed, it must be treated as an immutable, read-only data structure. Its primary state consists of different drop configurations (**BlockBreakingDropType**, **HarvestingDropType**, etc.) and the optimized map of tool-specific behaviors.

-   **Thread Safety:** This class is **not thread-safe**. It is designed to be constructed and fully initialized on a single thread (typically the main server thread) during the asset loading bootstrap sequence. After this phase, its immutable nature allows it to be safely read by multiple concurrent game-loop threads processing player actions.

    **WARNING:** Any attempt to access or modify a **BlockGathering** instance from another thread while assets are still being loaded will result in undefined behavior and likely race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.BlockGathering | O(1) | Serializes a subset of the gathering configuration into a network packet for client synchronization. |
| getBreaking() | BlockBreakingDropType | O(1) | Retrieves the configuration for drops when a block is broken by any means. |
| getHarvest() | HarvestingDropType | O(1) | Retrieves the configuration for drops when a block is harvested, typically with a specific tool like shears. |
| getToolData() | Map<String, BlockToolData> | O(1) | Returns the optimized map of tool-specific drop behaviors, keyed by tool type ID. |
| shouldUseDefaultDropWhenPlaced() | boolean | O(1) | Returns a flag indicating if player-placed blocks should ignore drop lists and use default behavior. |

## Integration Patterns

### Standard Usage

**BlockGathering** is not retrieved directly. It is accessed through its parent **BlockType** asset to make gameplay decisions. A system responsible for block interactions would query this object to determine the correct outcome.

```java
// Example from a hypothetical BlockInteractionSystem
void handleBlockBreak(Player player, Block block) {
    BlockType blockType = block.getType();
    BlockGathering gatheringConfig = blockType.getGatheringConfig();

    if (gatheringConfig == null) {
        // No special gathering rules, proceed with default logic
        return;
    }

    Item heldItem = player.getHeldItem();
    Map<String, BlockToolData> toolData = gatheringConfig.getToolData();

    if (heldItem != null && toolData.containsKey(heldItem.getTypeId())) {
        // A specific tool is being used, calculate drops from its drop list
        BlockToolData specificToolRule = toolData.get(heldItem.getTypeId());
        spawnDropsFromList(specificToolRule.getDropListId());
    } else {
        // Use the default breaking drop configuration
        BlockBreakingDropType breakingRule = gatheringConfig.getBreaking();
        spawnDropsFromList(breakingRule.getDropListId());
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new BlockGathering()`. This bypasses the **Codec** deserialization and, critically, the `afterDecode` logic that populates the `toolData` map. An object created this way will be in an incomplete and invalid state.
-   **State Modification:** Do not attempt to modify the state of a **BlockGathering** object after it has been loaded. It is designed as a read-only snapshot of the configuration file.

## Data Pipeline

The flow of data for **BlockGathering** begins with a static asset file and ends with its use in real-time game logic.

> **Asset Loading Flow:**
> BlockType JSON File -> Server Asset Loader -> **BlockGathering.CODEC** -> **BlockGathering Instance** (in memory) -> Attached to parent **BlockType**

> **Gameplay Logic Flow:**
> Player Action -> BlockInteractionSystem -> Get **BlockType** -> Get **BlockGathering Instance** -> Calculate Drops

> **Network Synchronization Flow:**
> Client Join -> Server Sync Process -> **BlockGathering Instance**.toPacket() -> Network Packet -> Client Asset System

