---
description: Architectural reference for SoftBlockDropType
---

# SoftBlockDropType

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Configuration Model

## Definition
```java
// Signature
public class SoftBlockDropType implements NetworkSerializable<SoftBlock> {
```

## Architecture & Concepts
The SoftBlockDropType class is a server-side data model that defines the drop behavior and breakability of a specific block type. It is not a service or manager, but rather a static data container deserialized from game asset files.

Its primary role is to act as a bridge between the asset configuration system and the runtime game logic. The static CODEC field is the core of this mechanism, providing the Hytale asset pipeline with the rules to decode a block's drop configuration (likely from a JSON file) into a strongly-typed Java object.

By implementing the NetworkSerializable interface, this class also defines its public contract for network replication. The server uses the toPacket method to convert this detailed configuration into a lightweight SoftBlock packet, which is then sent to clients. This ensures clients have the necessary information to predict block-breaking outcomes without possessing the full server-side asset configuration.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale asset loading system during server initialization. The system uses the static BuilderCodec instance, SoftBlockDropType.CODEC, to deserialize asset files into memory. Manual instantiation is a design violation.
- **Scope:** An instance of SoftBlockDropType is scoped to the block type it defines. These objects are loaded once and cached in a central registry, persisting for the entire duration of the server session. They are effectively singletons per block definition.
- **Destruction:** Instances are dereferenced and marked for garbage collection only when the server shuts down and its asset registries are cleared.

## Internal State & Concurrency
- **State:** This class is designed to be **effectively immutable**. Although its fields are not declared as final, they are populated once during asset deserialization and are not intended to be modified at runtime. The withoutDrops method reinforces this pattern by returning a *new* instance rather than mutating the existing one.
- **Thread Safety:** As a de-facto immutable data holder, this class is **thread-safe for reads**. Game logic from any thread can safely access its properties without synchronization, provided the anti-pattern of runtime mutation is avoided. The object contains no internal locks.

## API Surface
The public API is focused on data access and network serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | SoftBlock | O(1) | Serializes the object's state into a network-ready SoftBlock packet. |
| getItemId() | String | O(1) | Returns the asset identifier for the item required to efficiently break the block. |
| getDropListId() | String | O(1) | Returns the asset identifier for the ItemDropList used to generate drops. |
| isWeaponBreakable() | boolean | O(1) | Returns true if the block can be broken by a weapon. |
| withoutDrops() | SoftBlockDropType | O(1) | Creates a new, distinct instance with null drop and item identifiers. |

## Integration Patterns

### Standard Usage
Developers should not interact with this class directly. Instead, game systems retrieve it from a higher-level BlockType configuration object to make decisions about game mechanics, such as block breaking and loot generation.

```java
// Hypothetical engine code for handling a block break event
void onBlockBroken(Player player, Block brokenBlock) {
    BlockType type = brokenBlock.getType();
    SoftBlockDropType dropConfig = type.getSoftBlockDropConfig();

    if (dropConfig == null) {
        // Not a soft block or has no special drop rules
        return;
    }

    // Use the config to find the correct drop list asset
    ItemDropList dropList = AssetManager.get(ItemDropList.class, dropConfig.getDropListId());
    if (dropList != null) {
        world.spawnDrops(brokenBlock.getPosition(), dropList.generateDrops());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new SoftBlockDropType()`. All configuration must be defined in and loaded from the server's asset files to ensure data integrity and consistency.
- **Runtime Mutation:** Do not modify the fields of a SoftBlockDropType instance retrieved from an asset cache. This is not thread-safe and will lead to unpredictable, global side effects for all blocks of that type.

## Data Pipeline
The primary data flow for this class is from a configuration file on disk to an in-memory object, which is then serialized for network transmission to game clients.

> Flow:
> Block Asset File (e.g., JSON) -> Asset Loader -> **SoftBlockDropType.CODEC** -> **SoftBlockDropType Instance** -> toPacket() -> SoftBlock Network Packet -> Client Game State

