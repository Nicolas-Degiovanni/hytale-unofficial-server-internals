---
description: Architectural reference for ItemGlider
---

# ItemGlider

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Data Object

## Definition
```java
// Signature
public class ItemGlider implements NetworkSerializable<com.hypixel.hytale.protocol.ItemGlider> {
```

## Architecture & Concepts
The ItemGlider class is a server-side configuration data model that defines the physics and behavior of a glider item. It is not a service or an active component; rather, it is a passive data structure populated by the engine's asset loading system.

Its primary architectural role is to serve as a strongly-typed representation of data defined in external asset files (e.g., JSON). The static **CODEC** field is the cornerstone of this design. It provides a declarative schema that maps configuration keys, such as *TerminalVelocity*, to the corresponding fields within the class. This pattern decouples the game logic from the raw data format, allowing designers to tune glider behavior without modifying source code.

Furthermore, by implementing the NetworkSerializable interface, this class acts as a bridge between the server's internal configuration state and the client-facing network protocol. The `toPacket` method translates this server-side configuration object into a lightweight, optimized data structure suitable for transmission to game clients.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale asset pipeline during server initialization or asset hot-reloading. The static `BuilderCodec` is responsible for instantiating and populating the object by parsing the corresponding item asset file. Direct manual instantiation is an unsupported and dangerous operation.
- **Scope:** An ItemGlider instance has a singleton-like scope relative to its parent item asset. It is loaded once and persists in memory for the entire server session, cached within a central asset registry.
- **Destruction:** The object is marked for garbage collection only when its parent asset is unloaded from the asset registry, which typically occurs during a server shutdown or a full asset refresh.

## Internal State & Concurrency
- **State:** The object's state is mutable during the deserialization process managed by the `CODEC`. After this initial population, it is intended to be treated as immutable configuration data for the remainder of its lifecycle.
- **Thread Safety:** **This class is not thread-safe.** It contains no internal locking mechanisms. It is designed to be populated on a dedicated asset-loading thread and subsequently read by multiple game logic threads. Any external mutation of its fields after initialization will lead to race conditions and unpredictable game state.

## API Surface
The public API is minimal, primarily exposing the configuration data and the network serialization mechanism.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | static BuilderCodec | O(N) | Defines the declarative schema for deserializing the object from an asset file. N is the number of fields. |
| toPacket() | ItemGlider (protocol) | O(1) | Creates and returns a network-ready packet representation of the configuration. |

## Integration Patterns

### Standard Usage
This class is not meant to be retrieved or used directly. It is a component of a larger item asset. Game logic interacts with it by first obtaining the parent item asset and then accessing its glider configuration component.

```java
// Hypothetical example of accessing glider properties from a game system
ItemAsset gliderItem = AssetSystem.get("hytale:wooden_glider");
ItemGlider gliderConfig = gliderItem.getComponent(ItemGlider.class);

if (gliderConfig != null) {
    float maxSpeed = gliderConfig.getSpeed();
    // ... apply physics based on configuration
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ItemGlider()`. The object will be uninitialized and will cause NullPointerExceptions in game systems. The asset system is the sole owner of its lifecycle.
- **Post-Load Mutation:** Modifying the fields of a cached ItemGlider instance after it has been loaded is a critical error. This will create inconsistent behavior for all players and systems interacting with that item type.

## Data Pipeline
The ItemGlider class is a key stage in the pipeline that transforms static configuration files into live game data and network packets.

> Flow:
> Item Asset File (JSON) -> Asset Loading Service -> **ItemGlider.CODEC** -> **ItemGlider Instance** (In-Memory Cache) -> Game Physics System -> `toPacket()` -> Network Serialization Layer -> Client Game Instance

