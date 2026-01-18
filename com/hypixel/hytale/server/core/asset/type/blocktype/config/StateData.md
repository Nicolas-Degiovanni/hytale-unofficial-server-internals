---
description: Architectural reference for StateData
---

# StateData

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Configuration Data Object

## Definition
```java
// Signature
public class StateData {
```

## Architecture & Concepts
The StateData class is a data-driven configuration object that defines the relationship between abstract block states and concrete block assets. It acts as a mapping layer, translating a named state, such as *facing=north*, into a specific asset key that the engine can load, for example, *my_mod:oak_stairs_north*.

This class is a fundamental component of Hytale's block variant system. It is not intended for direct instantiation by developers. Instead, instances are created exclusively by the engine's asset loading pipeline through a sophisticated **Codec** system. The class defines its own deserialization logic via static `BuilderCodec` and `CodecMapCodec` fields, allowing it to be populated directly from structured data files (e.g., JSON) that are part of a game asset pack.

Its primary role is to hold the bidirectional mapping between state names and the corresponding block asset keys, and to provide a mechanism for converting this mapping into a network-efficient format for client-server communication.

### Lifecycle & Ownership
- **Creation:** Instances are created and hydrated by the `StateData.CODEC` during the server's asset loading phase. The constructor is invoked via reflection by the codec, and fields are populated from the source asset file. A critical `afterDecode` hook is registered to perform post-processing, such as generating the reverse `blockToState` map for efficient lookups.
- **Scope:** The lifetime of a StateData object is strictly bound to its parent `BlockType` asset. It is loaded when the `BlockType` is loaded and remains in memory for the duration of the server session or until the asset is explicitly unloaded.
- **Destruction:** The object is eligible for garbage collection once its owning `BlockType` is unloaded by the AssetManager. There is no manual destruction or cleanup method.

## Internal State & Concurrency
- **State:** The internal state is mutable during the asset deserialization process. After the `afterDecode` hook completes, the `blockToState` map is made unmodifiable. However, the primary `stateToBlock` map and the `copyFrom` method mean the object is not strictly immutable. **WARNING:** It should be treated as a read-only object after the initial asset loading phase is complete.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal locks or synchronization primitives. It is designed to be instantiated on a background asset-loading thread and subsequently accessed for reads from the main server thread. Any concurrent modification from multiple threads will result in data corruption and undefined behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | String | O(1) | Returns the unique identifier for this state group. |
| getBlockForState(state) | String | O(1) | Retrieves the block asset key associated with the given state name. |
| getStateForBlock(blockTypeKey) | String | O(1) | Retrieves the state name associated with the given block asset key. |
| toPacket(current) | Map | O(N) | Transforms the state map into a network-serializable format. N is the number of states. Returns a map of state names to integer asset indices. |
| copyFrom(state) | void | O(1) | Performs a shallow copy of map references from another StateData instance. **WARNING:** This is a mutating operation. |

## Integration Patterns

### Standard Usage
A developer will almost never interact with this class directly. Instead, it is accessed through a loaded `BlockType` asset to resolve block variants based on their state.

```java
// Assume 'blockType' is a fully loaded asset
StateData stateData = blockType.getStateData();

// Get the specific block asset for the "north" facing state
String northBlockKey = stateData.getBlockForState("north");

// Get the state name for a specific block asset
String stateName = stateData.getStateForBlock("hytale:stone_stairs_east");
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new StateData()`. This bypasses the codec system and results in a useless, uninitialized object. The internal maps will be null, leading to `NullPointerException` at runtime.
- **Post-Initialization Mutation:** Do not call `copyFrom` or modify the map returned by `stateToBlock` after an asset has been loaded and is in use by the server. This can lead to inconsistent state across the application.
- **Cross-Thread Writes:** Do not modify a StateData object from any thread after it has been made available to the main game loop. All initialization must be completed on the asset loading thread.

## Data Pipeline
The StateData object is a product of the server's asset processing pipeline. Data flows from a raw asset file on disk, through the codec system, and becomes part of a larger, in-memory asset representation.

> Flow:
> Block Asset File (JSON) -> AssetManager -> **StateData.CODEC** -> **StateData Instance** (Hydration & Post-Processing) -> `BlockType` Asset -> Game Logic / Network Serialization (`toPacket`)

