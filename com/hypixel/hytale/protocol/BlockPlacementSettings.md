---
description: Architectural reference for BlockPlacementSettings
---

# BlockPlacementSettings

**Package:** com.hypixel.hytale.protocol
**Type:** Data Structure / DTO

## Definition
```java
// Signature
public class BlockPlacementSettings {
```

## Architecture & Concepts
BlockPlacementSettings is a fundamental data structure within the Hytale network protocol layer. It is not a service or manager, but rather a Plain Old Java Object (POJO) that serves as a Data Transfer Object (DTO). Its primary responsibility is to encapsulate the complete set of rules and behaviors governing how a specific block can be placed in the world by a player.

This class is meticulously designed for high-performance network communication. Its fixed 16-byte memory layout, exposed through constants like FIXED_BLOCK_SIZE, is a critical design choice. This allows for zero-overhead serialization and deserialization directly to and from Netty ByteBufs, avoiding costly reflection or intermediate data formats.

Architecturally, BlockPlacementSettings acts as a data contract between the server and client. It ensures both parties agree on placement logic, such as rotation behavior, attachment rules (wall, floor, ceiling), and visual previews. These settings are typically defined within game asset files and are transmitted to the client when necessary, for example, when a player equips a block item in their hotbar.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary circumstances:
    1.  By the network layer when a packet containing these settings is received, via the static `deserialize` factory method.
    2.  By higher-level game logic systems, such as an AssetManager or ItemFactory, which might load these settings from a block's definition file and instantiate this class to represent them in memory.
- **Scope:** BlockPlacementSettings objects are transient and short-lived. An instance typically exists only for the duration of a specific task, such as processing a single network packet or handling one player interaction. They are not session-scoped and are not intended to be long-lived objects.
- **Destruction:** The object is managed by the Java Garbage Collector. There are no native resources or manual cleanup procedures. Once all references to an instance are released, it becomes eligible for garbage collection.

## Internal State & Concurrency
- **State:** The internal state is fully **mutable**. All fields are public, allowing for direct modification. This design prioritizes performance and ease of use within a controlled, single-threaded context, but it carries significant risks if misused.
- **Thread Safety:** This class is **not thread-safe**. Its public mutable fields make it inherently unsafe for concurrent access. If an instance is read by one thread while being modified by another, the reading thread may observe a corrupt or inconsistent state.

**WARNING:** Never share a BlockPlacementSettings instance between threads. If settings must be passed to another thread, a defensive copy must be created using the `clone()` method or the copy constructor.

## API Surface
The public contract is dominated by static utility methods for serialization and validation, reflecting its role as a network data structure.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static BlockPlacementSettings | O(1) | Constructs a new instance by reading 16 bytes from the given buffer at the specified offset. |
| serialize(ByteBuf) | void | O(1) | Writes the instance's 16-byte representation into the provided buffer. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a bounds check to ensure the buffer contains enough data to deserialize the object. |
| computeSize() | int | O(1) | Returns the fixed size of the serialized data, which is always 16. |

## Integration Patterns

### Standard Usage
The canonical use case involves deserializing the object from a network buffer within a packet handler, then passing the resulting immutable data to a game logic system for processing.

```java
// Within a Netty channel handler or packet processor
public void handlePacket(ByteBuf packetData) {
    // Assume the settings start at offset 12 in this hypothetical packet
    ValidationResult result = BlockPlacementSettings.validateStructure(packetData, 12);
    if (result.isError()) {
        // Handle malformed packet
        return;
    }

    BlockPlacementSettings settings = BlockPlacementSettings.deserialize(packetData, 12);

    // Pass the fully-formed object to the game engine
    gameWorld.getPlayer().getActiveItem().setPlacementSettings(settings);
}
```

### Anti-Patterns (Do NOT do this)
- **Shared Mutable State:** Never store a single BlockPlacementSettings instance in a global or static context where multiple threads can access it. This will inevitably lead to race conditions.
    ```java
    // DO NOT DO THIS
    public static BlockPlacementSettings GLOBAL_SETTINGS = new BlockPlacementSettings();

    // Thread A
    GLOBAL_SETTINGS.rotationMode = BlockPlacementRotationMode.Free;

    // Thread B (may see an inconsistent state)
    if (GLOBAL_SETTINGS.rotationMode == BlockPlacementRotationMode.FacingPlayer) {
       // ...
    }
    ```
- **Manual Field-by-Field Construction:** While the default constructor is public, manually populating the object is error-prone. The data should originate from a trusted source, like network deserialization or an asset loader, to ensure correctness.

## Data Pipeline
BlockPlacementSettings primarily functions as a data payload within the network pipeline. It represents the deserialized, structured form of a raw byte sequence.

> Flow:
> Network ByteBuf -> `BlockPlacementSettings.deserialize` -> **BlockPlacementSettings Instance** -> Game Logic (e.g., Player Action System) -> Discarded

