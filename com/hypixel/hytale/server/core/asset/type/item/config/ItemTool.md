---
description: Architectural reference for ItemTool
---

# ItemTool

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Transient Configuration Model

## Definition
```java
// Signature
public class ItemTool implements NetworkSerializable<com.hypixel.hytale.protocol.ItemTool> {
```

## Architecture & Concepts

The ItemTool class is a server-side data model that defines the specific behaviors and properties of an item when it is used as a tool (e.g., a pickaxe, axe, or shovel). It is not an active game object but rather a static configuration blueprint loaded from asset files.

This class is a critical component of the asset system, acting as a data container deserialized from JSON or a similar configuration format. Its primary architectural role is to bridge declarative game design (in asset files) with imperative game logic (in the server engine). It encapsulates all tool-related mechanics, such as mining speed, durability rules, and sound effect triggers.

The static **CODEC** field is the cornerstone of this class. It uses the engine's powerful `BuilderCodec` system to define the schema for deserializing the asset file. This codec handles field mapping, type conversion, validation, and lifecycle callbacks like `afterDecode`.

Furthermore, ItemTool implements the `NetworkSerializable` interface. This signifies its dual role: it is not only a server-side configuration but also the source for a network-optimized packet (`com.hypixel.hytale.protocol.ItemTool`) that is sent to clients. This ensures clients have the necessary information to predict tool behavior and render effects without needing access to the full server-side asset files.

## Lifecycle & Ownership

-   **Creation:** ItemTool instances are never created directly using the `new` keyword in game logic. They are instantiated exclusively by the `BuilderCodec` during the server's asset loading phase at startup. The engine scans for item asset files, and the `ItemTool.CODEC` is invoked to parse the relevant tool configuration section and construct the object.
-   **Scope:** An instance of ItemTool persists for the entire server session. Once loaded, it is held in memory by a higher-level `Item` asset configuration, which is in turn managed by a central asset registry. It is treated as immutable configuration data after the initial loading and processing is complete.
-   **Destruction:** The object is eligible for garbage collection only when the server shuts down and the central asset registries are cleared.

## Internal State & Concurrency

-   **State:** The object's state is mutable only during the asset deserialization process managed by the `BuilderCodec`. The `afterDecode` hook, which calls `processConfig`, is the final step in this mutation, transforming string-based asset identifiers (e.g., `hitSoundLayerId`) into more efficient, integer-based indices (e.g., `hitSoundLayerIndex`). After this point, the object should be considered a read-only data source. The `transient` index fields are a form of performance-oriented caching to avoid repeated string lookups in the game loop.

-   **Thread Safety:** This class is **not thread-safe**. It is designed to be created, populated, and processed on a single thread during server initialization. After initialization, it can be safely read by multiple game logic threads, provided no modifications are made. Any runtime modification would be a severe anti-pattern and would lead to unpredictable behavior and race conditions. The lazy initialization within the nested `DurabilityLossBlockTypes` class is also not thread-safe and relies on being called from a single, predictable context after asset loading.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.ItemTool | O(N) | Serializes the configuration into a network packet for the client. N is the number of tool specs. |
| getSpecs() | ItemToolSpec[] | O(1) | Returns the array of specifications defining tool effectiveness against material types. |
| getSpeed() | float | O(1) | Returns the base mining speed modifier for this tool. |
| getDurabilityLossBlockTypes() | DurabilityLossBlockTypes[] | O(1) | Returns the rules for how durability is lost when interacting with specific blocks. |
| getHitSoundLayerIndex() | int | O(1) | Returns the pre-calculated asset index for the sound to play on a successful block hit. |
| getIncorrectMaterialSoundLayerIndex() | int | O(1) | Returns the pre-calculated asset index for the sound to play when hitting an incorrect material. |

## Integration Patterns

### Standard Usage

This object is not retrieved directly. It is accessed through a parent `Item` configuration object, which is fetched from a central registry. Game logic then uses this data to perform calculations.

```java
// Pseudo-code demonstrating typical access
Item pickaxeItem = AssetRegistries.ITEMS.get("my_diamond_pickaxe");
ItemTool toolConfig = pickaxeItem.getToolComponent();

if (toolConfig != null) {
    float miningSpeed = toolConfig.getSpeed();
    // ... use miningSpeed in block breaking calculations
    int soundIndex = toolConfig.getHitSoundLayerIndex();
    // ... play sound using the resolved index
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new ItemTool()`. The object will be uninitialized, lack processed data like sound indices, and will not be part of the game's asset ecosystem. All creation must be handled by the asset loader via the `CODEC`.
-   **Runtime Modification:** Do not modify any fields of this object after it has been loaded. The server's state will become desynchronized from the asset files, and because the object is shared, changes will have unpredictable and widespread side effects.
-   **Premature Access:** Do not attempt to access the integer index fields (e.g., `hitSoundLayerIndex`) before the `processConfig` method has been executed by the `BuilderCodec`. They will contain their default value (0), not the correct, resolved asset index.

## Data Pipeline

The data for an ItemTool follows a clear path from a human-readable file to a compact network message.

> Flow:
> JSON Asset File on Disk -> Server Asset Loader -> **BuilderCodec** (Deserialization) -> **ItemTool** (In-Memory Model) -> `processConfig()` (Data Linking & Caching) -> `toPacket()` (Serialization) -> `com.hypixel.hytale.protocol.ItemTool` (Network Packet) -> Client Game Engine

