---
description: Architectural reference for ViewBobbing
---

# ViewBobbing

**Package:** com.hypixel.hytale.builtin.adventure.camera.asset.viewbobbing
**Type:** Configuration Asset

## Definition
```java
// Signature
public class ViewBobbing
   implements NetworkSerializable<com.hypixel.hytale.protocol.ViewBobbing>,
   JsonAssetWithMap<MovementType, AssetMap<MovementType, ViewBobbing>> {
```

## Architecture & Concepts
The ViewBobbing class is a data-driven configuration asset, not an active service. Its primary role is to define the properties of camera shake and movement (view bobbing) for a specific player `MovementType`, such as walking, sprinting, or sneaking.

This class is a foundational component of the Hytale Asset System. Its design is centered around the static `CODEC` field, an `AssetBuilderCodec`. This codec acts as a declarative blueprint that instructs the `AssetManager` on how to deserialize a corresponding JSON file into a fully-formed ViewBobbing Java object. It maps JSON keys, like "FirstPerson", directly to the fields of the class, such as the `firstPerson` `CameraShakeConfig`.

By implementing `NetworkSerializable`, this configuration can be synchronized from the server to the client. This ensures that all players experience consistent, server-defined camera effects, which is critical for competitive and cinematic gameplay. It effectively serves as a bridge between static JSON configuration on disk and the dynamic, runtime camera rendering system.

## Lifecycle & Ownership
- **Creation:** Instances are never instantiated directly via the `new` keyword in game logic. They are created exclusively by the Hytale `AssetManager` during the asset loading phase. The `AssetManager` uses the static `CODEC` to parse a JSON asset and populate a new ViewBobbing instance.
- **Scope:** The lifetime of a ViewBobbing object is strictly tied to the asset cache. It persists in memory as long as the asset pack it belongs to remains loaded. For all practical purposes, it should be treated as a read-only singleton for its given `MovementType` within the asset registry.
- **Destruction:** The object is marked for garbage collection when the `AssetManager` unloads or reloads the associated assets, for example, when a player leaves a server or changes resource packs.

## Internal State & Concurrency
- **State:** Effectively immutable. After the `AssetManager` populates its fields from JSON during creation, its state is not designed to change. It serves as a read-only container for camera effect parameters.
- **Thread Safety:** This class is inherently thread-safe. Its immutable nature post-initialization allows it to be safely read from any thread without synchronization primitives. The rendering thread, for instance, can access this configuration data without risk of race conditions or data corruption from other game logic threads.

## API Surface
The public API is minimal, focusing on data access and network serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | MovementType | O(1) | Returns the specific movement type this configuration applies to. |
| toPacket() | com.hypixel.hytale.protocol.ViewBobbing | O(1) | Serializes the configuration into a network packet for client-server synchronization. Allocates a new packet object. |

## Integration Patterns

### Standard Usage
Developers should not interact with individual ViewBobbing instances directly. Instead, they should retrieve the complete map of all view bobbing profiles from the `AssetManager` and select the appropriate one based on the entity's current state.

```java
// System-level code to retrieve and apply the correct configuration
AssetMap<MovementType, ViewBobbing> allBobbingProfiles = assetManager.get("hytale:camera_viewbobbing");

// In the player update loop, select the profile based on current movement
MovementType currentMovement = player.getMovementType();
ViewBobbing currentProfile = allBobbingProfiles.get(currentMovement);

// The profile can then be used by the network or rendering systems
if (currentProfile != null) {
    networkManager.sendPacketToClient(player, currentProfile.toPacket());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ViewBobbing()`. An object created this way will be uninitialized and incomplete, as it bypasses the critical deserialization logic defined in its `CODEC`. This will result in `NullPointerException` and other undefined behavior.
- **Runtime Modification:** Do not attempt to modify the fields of a ViewBobbing instance after it has been loaded. The system relies on this configuration being a stable, read-only source of truth. Modifying it at runtime can lead to visual inconsistencies and desynchronization between client and server.

## Data Pipeline
The ViewBobbing class is a destination point in the asset loading pipeline and a source of data for the rendering and network pipelines.

> **Asset Loading Flow:**
> JSON File on Disk -> AssetManager -> **AssetBuilderCodec** -> **ViewBobbing Instance** (In-Memory Cache)
>
> **Runtime Data Flow:**
> **ViewBobbing Instance** -> `toPacket()` -> Network Layer -> Client
>
> **ViewBobbing Instance** -> Camera Rendering System -> Final Rendered Frame

