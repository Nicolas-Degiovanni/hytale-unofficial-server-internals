---
description: Architectural reference for BlockBreakingDropType
---

# BlockBreakingDropType

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Data Model / Configuration Object

## Definition
```java
// Signature
public class BlockBreakingDropType implements NetworkSerializable<BlockBreaking> {
```

## Architecture & Concepts

The **BlockBreakingDropType** class is a server-side data model that defines the outcome of a block-breaking event. It is not a service or a manager, but rather a static configuration object deserialized from game asset files. Its primary role is to encapsulate the rules for what items, if any, are generated when a player successfully breaks a specific type of block.

This class acts as a critical bridge between the static asset configuration system and the dynamic, real-time networking layer. It implements the **NetworkSerializable** interface, which mandates a **toPacket** method. This contract signifies that a **BlockBreakingDropType** instance, which holds detailed server-side configuration, can be transformed into a more lightweight **BlockBreaking** protocol packet suitable for transmission to game clients.

The static **CODEC** field is the cornerstone of this class's integration into the engine's asset pipeline. This **BuilderCodec** defines the declarative mapping between keys in a JSON or HOCON asset file (e.g., "GatherType", "ItemId") and the fields of this Java object. The presence of **addValidatorLate** indicates a sophisticated two-pass asset loading strategy. The engine first loads the raw data, and only later validates references to other assets like **Item** and **ItemDropList**. This prevents deadlocks and resolves circular dependencies during the server's bootstrap sequence.

## Lifecycle & Ownership

-   **Creation:** Instances of **BlockBreakingDropType** are not created manually by developers using the *new* keyword. They are instantiated exclusively by the server's asset loading framework during startup. The static **CODEC** is used to deserialize the object from a parent block's configuration file.

-   **Scope:** The lifetime of a **BlockBreakingDropType** object is bound to the **BlockType** asset that defines it. It persists in memory for the entire duration of the server session, as long as the parent block asset remains loaded.

-   **Destruction:** These objects are managed by the Java Garbage Collector. They are eligible for cleanup only when their parent **BlockType** asset is unloaded, which typically occurs during a server shutdown or a hot-reload of game assets. There is no explicit destruction method.

## Internal State & Concurrency

-   **State:** This class is a mutable data-holding object. However, it should be treated as *effectively immutable* after the server's initial asset loading phase is complete. Its state is populated once during deserialization and is not intended to be modified at runtime. The **withoutDrops** method correctly promotes this pattern by returning a new instance rather than mutating the existing one.

-   **Thread Safety:** **BlockBreakingDropType** is **not thread-safe**. It contains no internal locking mechanisms or concurrent data structures. This design is intentional and performant, based on the assumption that the object will be read from concurrently by multiple game logic threads but never written to after its initial creation.

    **Warning:** Modifying an instance of this class from any thread after the server has started will lead to severe race conditions and unpredictable game behavior. All fields must be treated as read-only.

## API Surface

The public API is minimal, focusing on data transformation and access. Simple getters are omitted for brevity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | BlockBreaking | O(1) | Transforms this server-side configuration object into a network-optimized **BlockBreaking** packet for client transmission. |
| withoutDrops() | BlockBreakingDropType | O(1) | Creates and returns a new instance with identical gathering properties but with all drop information (item, quantity, droplist) removed. |

## Integration Patterns

### Standard Usage

This object is not retrieved directly. Instead, it is accessed through its parent **BlockType** asset to determine the rules for a block interaction.

```java
// Conceptual example of server-side game logic
BlockType targetBlock = world.getBlockTypeAt(position);
BlockBreakingDropType dropConfig = targetBlock.getDropConfigForTool(player.getHeldTool());

if (dropConfig != null) {
    // Use the configuration to spawn items or perform other logic
    spawnDrops(world, position, dropConfig);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new BlockBreakingDropType()`. The object's state is defined entirely by asset files. Manual creation bypasses the asset pipeline and will result in an incomplete or invalid object that is disconnected from the game's configuration.

-   **Runtime Modification:** Do not modify the fields of a **BlockBreakingDropType** instance after it has been loaded. This violates the "configuration-as-code" principle and is not thread-safe.

## Data Pipeline

The data for this object originates from asset files and is ultimately transformed into network packets during gameplay.

> **Asset Loading Flow:**
> Block Configuration File (e.g., stone.json) -> Server Asset Loader -> **BlockBreakingDropType.CODEC** -> In-Memory **BlockBreakingDropType** Instance

> **Gameplay Logic Flow:**
> Player breaks block -> Game Logic retrieves **BlockBreakingDropType** -> **toPacket()** method is called -> **BlockBreaking** Packet -> Network Layer -> Game Client

