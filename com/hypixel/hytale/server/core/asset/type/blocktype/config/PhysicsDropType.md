---
description: Architectural reference for PhysicsDropType
---

# PhysicsDropType

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Configuration Data Model

## Definition
```java
// Signature
public class PhysicsDropType {
```

## Architecture & Concepts
The PhysicsDropType class is a passive data structure that defines the item drops resulting from a physics-based event, such as a block breaking under gravity. It is not an active service but rather a component of the server's declarative configuration system.

This class acts as a data contract between game design configuration files (e.g., JSON) and the server's core systems. Its primary architectural feature is the static **CODEC** field, an instance of BuilderCodec. This codec integrates the class into the engine's universal asset serialization pipeline, allowing the server to load and validate drop behaviors directly from asset files without requiring code changes.

The design heavily relies on asset referencing. Instead of containing full Item or ItemDropList objects, it holds string identifiers (**itemId**, **dropListId**). The codec includes late-stage validators which ensure that these identifiers correspond to valid, loaded assets during a secondary pass of the asset loading process. This decouples the block configuration from the item configuration, promoting modularity and preventing circular dependencies during asset loading.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the engine's asset loading system via the static **CODEC** field. The codec is invoked during server bootstrap or asset hot-reloading to deserialize a portion of a block's configuration file into a PhysicsDropType object. Manual instantiation is a design violation.
- **Scope:** The lifetime of a PhysicsDropType instance is bound to its parent configuration object, typically a BlockType. It persists in memory for the entire server session once loaded.
- **Destruction:** The object is marked for garbage collection when the server's asset registry is cleared, which occurs on server shutdown or during a full asset reload.

## Internal State & Concurrency
- **State:** Effectively **immutable**. After deserialization, the internal state (**itemId**, **dropListId**) is not intended to be modified. The class functions as a read-only container for configuration data.
- **Thread Safety:** **Conditionally thread-safe**. Once an instance is fully constructed and published by the asset loading system, it can be safely read by multiple game logic threads simultaneously due to its immutable nature. The creation process itself, however, is part of the single-threaded asset loading pipeline and is not designed for concurrent invocation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getItemId() | String | O(1) | Returns the asset identifier for a single item to be dropped. |
| getDropListId() | String | O(1) | Returns the asset identifier for an ItemDropList, used for complex drop logic. |
| withoutDrops() | PhysicsDropType | O(1) | Returns null. This is a utility for configuration override systems to explicitly specify that no drops should occur. |

## Integration Patterns

### Standard Usage
This class is consumed by server-side game logic that handles block destruction. The system retrieves the block's configuration, accesses its PhysicsDropType, and uses the contained identifiers to query the appropriate item or loot table manager.

```java
// Conceptual example within a block physics system
BlockType blockConfig = getBlockConfiguration(blockId);
PhysicsDropType dropInfo = blockConfig.getPhysicsDropType();

if (dropInfo != null) {
    String itemId = dropInfo.getItemId();
    String dropListId = dropInfo.getDropListId();

    if (itemId != null) {
        // Spawn the single item
        world.spawnItem(itemId, blockPosition);
    } else if (dropListId != null) {
        // Process the drop list for more complex loot
        lootManager.spawnDropsFromList(dropListId, blockPosition);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new PhysicsDropType()`. The object's state is entirely dependent on the asset pipeline for correct initialization and validation. Direct creation bypasses this crucial process.
- **Runtime Mutation:** Do not attempt to modify the fields of a PhysicsDropType instance after it has been loaded. This will lead to unpredictable and inconsistent behavior across the server, as game systems expect configuration to be static.

## Data Pipeline
The flow of data from configuration to in-game effect is linear and centrally managed by the asset system.

> Flow:
> Block Asset File (JSON) -> AssetManager -> **PhysicsDropType.CODEC** -> In-memory **PhysicsDropType** instance -> Block Physics Engine -> Item/Loot Manager -> World Simulation

