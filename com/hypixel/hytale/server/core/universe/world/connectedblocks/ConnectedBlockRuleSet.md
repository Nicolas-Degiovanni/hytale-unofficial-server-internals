---
description: Architectural reference for ConnectedBlockRuleSet
---

# ConnectedBlockRuleSet

**Package:** com.hypixel.hytale.server.core.universe.world.connectedblocks
**Type:** Base Class / Strategy

## Definition
```java
// Signature
public abstract class ConnectedBlockRuleSet {
```

## Architecture & Concepts

The ConnectedBlockRuleSet is an abstract base class that defines the contract for implementing "connected block" logic. In voxel engines, this system is responsible for changing a block's appearance based on its adjacent neighbors. For example, it allows a series of glass blocks to form a seamless pane of glass, or for a fence post to connect to an adjacent fence gate.

This class embodies a **Strategy Pattern**. Each concrete implementation (e.g., GlassRuleSet, FenceRuleSet) encapsulates a unique algorithm for determining the correct block state. These rule sets are not instantiated dynamically during gameplay; they are defined as part of a block's asset data and loaded at startup.

The static `CODEC` field is the central mechanism for this data-driven design. It allows the asset loading system to deserialize different rule set implementations from configuration files by mapping a "Type" string (e.g., "type": "glass") to a specific Java class. This makes the system highly extensible for developers and modders, as new connection behaviors can be added without modifying the core world simulation engine.

### Lifecycle & Ownership
- **Creation:** Instances are created by the asset system during server bootstrap. The `CodecMapCodec` reads a block's configuration file, identifies the rule set type, and instantiates the corresponding subclass.
- **Scope:** A single rule set instance is typically shared by all blocks of a given type (e.g., one GlassRuleSet instance is used for all glass blocks). They are stateless strategies whose lifetime is bound to the server session.
- **Destruction:** Instances are garbage collected when the server shuts down or initiates a full asset reload.

## Internal State & Concurrency
- **State:** The base class is stateless. Subclasses are expected to be stateless or have their state be immutable after the initial asset loading phase. The `updateCachedBlockTypes` method provides a hook for subclasses to perform pre-computation and cache asset references for performance, but this cache must not be modified during gameplay.
- **Thread Safety:** Implementations of this class are **not** guaranteed to be thread-safe. The primary method, `getConnectedBlockType`, receives a `World` instance as an argument. All operations on the `World` object must be performed on the main world simulation thread or within a context that has acquired the necessary locks for the relevant world region. Calling this method from an asynchronous task without proper synchronization will lead to severe data corruption and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onlyUpdateOnPlacement() | boolean | O(1) | A policy flag. If true, the rule set is only evaluated when the block is first placed, not when its neighbors change. |
| getConnectedBlockType(...) | Optional | O(k) | The core evaluation method. Calculates the correct block state based on its neighbors. Complexity is relative to the number of neighbors checked. |
| updateCachedBlockTypes(...) | void | O(n) | A lifecycle hook called after asset loading to allow for pre-computation and caching of related block types. |
| toPacket(...) | com.hypixel.hytale.protocol.ConnectedBlockRuleSet | O(1) | Serializes the rule set's definition into a network packet for the client. This allows the client to perform predictive rendering. |

## Integration Patterns

### Standard Usage

A `BlockType` asset definition specifies its connection logic. The engine uses this definition to invoke the rule set during a block update.

```java
// Pseudo-code demonstrating the engine's internal flow
void onBlockUpdate(World world, Vector3i position) {
    BlockType currentType = world.getBlockTypeAt(position);
    ConnectedBlockRuleSet ruleSet = currentType.getConnectedBlockRuleSet();

    if (ruleSet != null) {
        Optional<ConnectedBlockResult> result = ruleSet.getConnectedBlockType(world, position, ...);
        if (result.isPresent()) {
            world.setBlockStateAt(position, result.get().getNewState());
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new GlassRuleSet()`. The system is designed to be data-driven. Rule sets must be registered with the `CODEC` and defined in asset files to ensure they are loaded and managed correctly by the engine.
- **Stateful Implementations:** Do not store per-block or per-chunk state within a rule set instance. These objects are shared singletons and must remain stateless to prevent concurrency issues and incorrect behavior.
- **Unsynchronized World Access:** Never invoke `getConnectedBlockType` from a separate thread without first acquiring a lock on the relevant `World` chunk. Doing so is a critical race condition.

## Data Pipeline

The rule set acts as a pure function within the world's block update pipeline. It takes world state as input and produces a new block state as output.

> Flow:
> Block Placement/Update Event -> World Simulation Tick -> **ConnectedBlockRuleSet.getConnectedBlockType()** -> New Block State -> World Data Update -> Network Packet to Clients

