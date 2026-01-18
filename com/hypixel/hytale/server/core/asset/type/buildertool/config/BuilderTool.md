---
description: Architectural reference for BuilderTool
---

# BuilderTool

**Package:** com.hypixel.hytale.server.core.asset.type.buildertool.config
**Type:** Configuration Asset

## Definition
```java
// Signature
public class BuilderTool implements JsonAssetWithMap<String, DefaultAssetMap<String, BuilderTool>>, NetworkSerializable<BuilderToolState> {
```

## Architecture & Concepts
The BuilderTool class serves as a static, data-driven template for all builder tools within the game server. It is not a live object representing a tool in a player's inventory; rather, it is the immutable schema or definition loaded from server asset files (e.g., JSON) that dictates a tool's properties, behaviors, and configurable arguments.

Its primary architectural role is to decouple the definition of a tool from its instance-specific data.
-   The **BuilderTool** asset defines the *what*: what arguments exist (e.g., "size", "shape"), their types, default values, and validation rules.
-   An **ItemStack** holds the *instance data*: the specific values a player has configured for their particular tool (e.g., size=5, shape=SQUARE), stored as BSON in the item's metadata.

This class provides the core logic for serializing and deserializing this instance data to and from an ItemStack. It acts as the authoritative source for a tool's capabilities, both for server-side logic and for synchronization with the client. The implementation of NetworkSerializable allows the tool's definition to be packaged into a BuilderToolState packet, enabling the client to dynamically render the correct UI for any given tool without having hardcoded knowledge of it.

## Lifecycle & Ownership
-   **Creation:** BuilderTool instances are created exclusively by the Hytale AssetRegistry during server bootstrap. The static CODEC field, an AssetBuilderCodec, is used to deserialize JSON asset files into fully-formed BuilderTool objects. Manual instantiation is strictly forbidden.
-   **Scope:** An instance of BuilderTool has a global, server-wide scope. A single instance exists for each unique tool ID (e.g., one for `hytale:fill_tool`, one for `hytale:sculpt_brush`). These instances are effectively singletons for their respective asset types and persist for the entire server session. They are shared and read by numerous systems.
-   **Destruction:** Instances are destroyed only during server shutdown when the AssetRegistry is cleared and all assets are unloaded.

## Internal State & Concurrency
-   **State:** The object's state is considered **immutable** after the initial asset loading phase. All definitional fields like id, isBrush, and the map of args are populated once by the codec and never change. The only piece of mutable state is the `cachedPacket` field, a SoftReference to a network packet. This is a thread-safe performance optimization to avoid repeated serialization.
-   **Thread Safety:** The class is **thread-safe for all read operations**, which constitute the vast majority of its use cases. Methods that appear to modify data, such as updateArgMetadata, are pure functions: they do not mutate the BuilderTool instance or the input ItemStack. Instead, they return a *new* ItemStack instance with the updated metadata. This functional approach avoids concurrency issues.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getActiveBuilderTool(Player) | static BuilderTool | O(1) | Utility to retrieve the BuilderTool asset corresponding to the item in a player's hand. Returns null if the item is not a builder tool. |
| getItemArgData(ItemStack) | ArgData | O(N) | Deserializes and returns the tool's instance-specific arguments from the ItemStack's metadata. Falls back to the tool's default values if no metadata is present. N is the number of arguments. |
| createItemStack(id, quantity, data) | ItemStack | O(N) | Creates a new ItemStack, serializing the provided ArgData into the item's BSON metadata. |
| updateArgMetadata(item, group, id, value) | ItemStack | O(N) | The primary mutation method. Validates a new value for an argument and returns a **new** ItemStack with updated metadata. Throws ToolArgException on validation failure. |
| toPacket() | BuilderToolState | O(N) / O(1) | Serializes the tool's static definition into a network packet for client synchronization. The result is cached in a SoftReference, making subsequent calls O(1). |

## Integration Patterns

### Standard Usage
The correct pattern involves retrieving the static BuilderTool asset and using it as a factory or manager for manipulating the instance data stored on an ItemStack.

```java
// How a developer should normally use this
Player player = ...;
ItemStack currentToolItem = player.getInventory().getItemInHand();
BuilderTool toolDefinition = BuilderTool.getActiveBuilderTool(player);

if (toolDefinition != null && currentToolItem != null) {
    // Read the current values from the item
    BuilderTool.ArgData currentArgs = toolDefinition.getItemArgData(currentToolItem);
    System.out.println("Current brush size: " + currentArgs.brush().size());

    // Update an argument and replace the item in the player's inventory
    try {
        ItemStack updatedToolItem = toolDefinition.updateArgMetadata(
            currentToolItem,
            BuilderToolArgGroup.Brush,
            "size",
            "10"
        );
        player.getInventory().setItemInHand(updatedToolItem);
    } catch (ToolArgException e) {
        player.sendMessage(e.getErrorMessage());
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BuilderTool()`. The server's AssetRegistry is the sole authority for creating and managing these objects. Direct instantiation will result in an uninitialized and non-functional object that is disconnected from the rest of the engine.
-   **State Mutation:** Do not attempt to modify the state of a BuilderTool instance after it has been loaded (e.g., via reflection). These assets are shared globally; mutating one would create unpredictable and catastrophic side effects across the entire server.
-   **Misunderstanding Ownership:** Do not store a reference to the `ArgData` object returned by `getItemArgData` and expect it to reflect future changes. It is a snapshot of the item's state at a moment in time. Always re-fetch it from the ItemStack when you need the current values.

## Data Pipeline
The BuilderTool class is central to two key data flows: asset loading and runtime item configuration.

**1. Asset Loading (Server Start)**
> Flow:
> JSON Asset File -> AssetRegistry -> **BuilderTool.CODEC** (Deserialization) -> In-Memory **BuilderTool** Instance

**2. Runtime Configuration & Client Sync**
> Flow:
> Player Input (Command/UI) -> Server Logic -> `BuilderTool.updateArgMetadata` -> New ItemStack with BSON Metadata -> Player Inventory -> `BuilderTool.toPacket` -> BuilderToolState Packet -> Network Layer -> Client UI Update

