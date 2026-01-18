---
description: Architectural reference for ItemAppearanceCondition
---

# ItemAppearanceCondition

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Configuration Data Object

## Definition
```java
// Signature
public class ItemAppearanceCondition implements NetworkSerializable<com.hypixel.hytale.protocol.ItemAppearanceCondition> {
```

## Architecture & Concepts

The ItemAppearanceCondition class is a data-driven configuration object that defines a specific visual and auditory state for an item based on a numeric condition. It is a fundamental component of Hytale's dynamic item appearance system, allowing designers to specify how an item's model, texture, particles, and sounds should change in response to game state, such as durability, charge level, or a custom variable.

This class acts as a bridge between static asset configuration (typically defined in JSON files) and the live, in-game representation of an item. It is not a service or a manager; it is a passive data container whose properties are read by other systems, such as the rendering engine or the item update logic.

The static **CODEC** field is the cornerstone of this class's design. It uses the Hytale Codec library to provide a declarative, robust, and version-safe mechanism for deserializing item configuration from disk. This approach centralizes all parsing, validation, and post-processing logic, ensuring that any instance of ItemAppearanceCondition is always in a valid state upon creation.

Furthermore, its implementation of the **NetworkSerializable** interface signifies its role in the server-client data flow. The server loads the complete configuration from assets, but serializes a more compact, network-optimized representation to the client using the toPacket method.

## Lifecycle & Ownership

-   **Creation:** Instances are exclusively created and hydrated by the asset loading system via the static **CODEC** field. This process occurs when the server boots or when assets are reloaded. The codec parses a corresponding block from an item's configuration file.
-   **Scope:** An ItemAppearanceCondition object does not exist in isolation. It is owned by a parent item asset configuration. Its lifetime is tied directly to the lifetime of its parent asset, which typically persists for the entire server session.
-   **Destruction:** The object is eligible for garbage collection when its parent item asset is unloaded, which usually happens during a server shutdown or a full asset reload. There is no manual destruction method.

## Internal State & Concurrency

-   **State:** The object's state is populated once during deserialization and is considered **effectively immutable** thereafter. All public methods are getters, providing read-only access to the configuration data.
    -   **Warning:** The class contains transient integer fields, such as worldSoundEventIndex, which are derived from string IDs during the `afterDecode` lifecycle hook. This is a load-time optimization to avoid repeated string-to-index lookups at runtime. These fields are critical for performance and are not part of the serialized asset format.

-   **Thread Safety:** The class is inherently **thread-safe for reads**. Because its state does not change after initial loading, multiple game systems (e.g., logic thread, audio thread) can safely access an instance without synchronization. The asset loading process that creates these objects is assumed to be a synchronized, single-threaded operation.

## API Surface

The public API is designed for read-only access to the configured properties.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCondition() | FloatRange | O(1) | Returns the numeric range for which this condition is active. |
| getModel() | String | O(1) | Returns the asset path for the model to use when the condition is met. |
| getTexture() | String | O(1) | Returns the asset path for the texture to apply. |
| getParticles() | ModelParticle[] | O(1) | Returns the array of particles to emit in a third-person view. |
| getWorldSoundEventIndex() | int | O(1) | Returns the pre-calculated integer index for the 3D sound event. |
| getLocalSoundEventIndex() | int | O(1) | Returns the pre-calculated integer index for the local sound event. |
| toPacket() | protocol.ItemAppearanceCondition | O(N) | Serializes the object into a network-optimized packet. Complexity is relative to the number of particles. |

## Integration Patterns

### Standard Usage

This object is not intended to be managed directly. Instead, game logic systems query the active ItemAppearanceCondition from a parent item based on its current state.

```java
// Conceptual example of how a system would use this object
// The Item class would hold a list of these conditions.

Item item = player.getHeldItem();
float currentDurability = item.getDurability();

// The Item's logic would find the first matching condition
ItemAppearanceCondition activeCondition = item.findConditionForValue(currentDurability);

if (activeCondition != null) {
    // The rendering system uses the condition's data
    renderer.setModel(activeCondition.getModel());
    renderer.setTexture(activeCondition.getTexture());
    
    // The audio system uses the pre-calculated index
    audioEngine.playSound(activeCondition.getWorldSoundEventIndex());
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ItemAppearanceCondition()`. This bypasses the CODEC, which is responsible for all validation and crucial post-processing steps like populating the sound event indices. An object created this way will be in an incomplete and invalid state.
-   **State Mutation:** Do not attempt to modify the state of a loaded ItemAppearanceCondition using reflection or other means. The entire asset system relies on these objects being immutable after creation. Unpredictable visual and auditory glitches will occur.
-   **Manual Serialization:** Do not write custom serialization logic. The toPacket method is the only supported way to prepare this object for network transmission, as it correctly handles the translation of data from the server-side representation to the client-side protocol.

## Data Pipeline

The data for an ItemAppearanceCondition flows from static configuration files on disk to the game client, undergoing several transformations.

> Flow:
> Item Asset JSON File -> Server Asset Loader -> **ItemAppearanceCondition.CODEC** -> In-Memory ItemAppearanceCondition Instance -> `toPacket()` Serialization -> Network Packet -> Game Client Rendering

---

