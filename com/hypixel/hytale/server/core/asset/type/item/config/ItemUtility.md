---
description: Architectural reference for ItemUtility
---

# ItemUtility

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Configuration Model

## Definition
```java
// Signature
public class ItemUtility implements NetworkSerializable<com.hypixel.hytale.protocol.ItemUtility> {
```

## Architecture & Concepts

The ItemUtility class is a data-centric configuration model that encapsulates the functional properties of a game item, such as its stat modifiers and usability flags. It is not a service or a manager; rather, it is a passive data structure that represents a deserialized portion of an item's asset definition file.

The core architectural concept of this class is its **two-phase hydration process**. During asset loading, the Hytale Codec system first populates the class with *raw*, string-based identifiers read directly from the configuration file (e.g., `rawStatModifiers`). In a second, internal step within the codec's `afterDecode` hook, these raw values are resolved into a more performant, runtime-ready format using integer-based identifiers (e.g., `statModifiers`).

This design pattern decouples the human-readable asset definitions from the server's high-performance runtime representation. The class also implements NetworkSerializable, indicating its role as a server-side data authority that can be serialized into a network packet and transmitted to the client.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the static `CODEC` field during the server's asset loading phase. The `BuilderCodec` acts as the sole factory, ensuring that the critical `afterDecode` logic is always executed. Direct instantiation is an anti-pattern and will result in an uninitialized, non-functional object. A static `DEFAULT` instance exists for fallback scenarios.
- **Scope:** An ItemUtility instance lives as long as its parent item asset is registered in the server's asset manager. It is effectively a session-scoped singleton tied to a specific item definition.
- **Destruction:** The object is marked for garbage collection when the server shuts down or performs a full asset reload, at which point all references from the central asset registry are cleared. It has no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The object's state is mutable only during the brief window of its creation and hydration by the `CODEC`. After the `afterDecode` hook completes, the object is treated as **effectively immutable**. All public access is read-only, and its internal state is not designed to be changed during the game loop.
- **Thread Safety:** This class is **not thread-safe** for writing. It is designed to be created and populated on a single thread during server initialization. However, because it is treated as immutable post-initialization, it is **safe for concurrent reads** from multiple game threads (e.g., entity processing threads) without requiring locks.

## API Surface

The public API is minimal, exposing only the resolved, runtime-ready data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isUsable() | boolean | O(1) | Returns true if the item has a utility function that can be triggered by a player. |
| isCompatible() | boolean | O(1) | Returns a generic compatibility flag, its purpose defined by consuming systems. |
| getStatModifiers() | Int2ObjectMap | O(1) | Returns the resolved map of stat modifiers, keyed by integer stat IDs. Returns null if none. |
| getEntityStatsToClear() | int[] | O(1) | Returns the resolved array of integer stat IDs that this item clears upon use. |
| toPacket() | ItemUtility | O(N) | Serializes the object into its network protocol equivalent for client transmission. N is the number of stat modifiers. |

## Integration Patterns

### Standard Usage

This class is not intended for direct instantiation. Game logic systems should retrieve it from a pre-loaded item asset definition.

```java
// Example: Retrieving ItemUtility from an Item asset
Item myItemAsset = assetRegistry.get(Item.class, "my_namespace:my_sword");
ItemUtility utilityConfig = myItemAsset.getUtility();

if (utilityConfig != null && utilityConfig.isUsable()) {
    // Apply stat modifiers to an entity
    Int2ObjectMap<StaticModifier[]> modifiers = utilityConfig.getStatModifiers();
    if (modifiers != null) {
        playerEntity.getStats().addModifiers(modifiers);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ItemUtility()`. This bypasses the `CODEC` deserialization and the critical `afterDecode` resolution logic. The resulting object will lack resolved stat modifiers and will be in an invalid state.
- **State Mutation:** Do not attempt to modify the state of an ItemUtility object after it has been loaded. The game engine relies on this configuration being stable and read-only throughout the server session.

## Data Pipeline

The creation and hydration of an ItemUtility object follows a strict, one-way data flow managed by the asset loading system.

> Flow:
> Item Asset File (JSON) -> Hytale Codec System -> **ItemUtility (Raw State)** -> `afterDecode` Hook -> **ItemUtility (Resolved State)** -> Game Systems (Read-Only) -> `toPacket()` -> Network Layer

