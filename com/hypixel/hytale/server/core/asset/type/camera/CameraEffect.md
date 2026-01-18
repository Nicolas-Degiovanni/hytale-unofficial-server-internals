---
description: Architectural reference for CameraEffect
---

# CameraEffect

**Package:** com.hypixel.hytale.server.core.asset.type.camera
**Type:** Data Model / Asset Type

## Definition
```java
// Signature
public abstract class CameraEffect implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, CameraEffect>> {
```

## Architecture & Concepts
The CameraEffect class is an abstract base type for all data-driven camera effects within the Hytale engine, such as screen shakes. It is not a service or manager, but rather a data model that represents the *definition* of a single, loadable asset.

Its primary architectural role is to serve as a bridge between configuration data (defined in JSON files) and the network protocol. By implementing the JsonAssetWithMap interface, CameraEffect signals its participation in the engine's core asset loading system. During startup, the AssetStore deserializes JSON definitions into concrete CameraEffect instances using the provided static CODEC.

At runtime, game logic retrieves a specific CameraEffect instance from the global AssetStore and uses it as a factory to generate a CameraShakeEffect network packet. This design decouples the game logic that *triggers* an effect from the data that *defines* the effect, allowing designers to create, modify, and balance camera effects without recompiling the engine.

The nested static class, MissingCameraEffect, implements the Null Object Pattern. If the AssetStore is queried for a non-existent effect ID, it returns an instance of this class. This prevents null pointer exceptions and allows the game to proceed without interruption, albeit without the intended visual effect.

## Lifecycle & Ownership
- **Creation:** CameraEffect instances are not instantiated directly using the new keyword. They are created exclusively by the AssetStore during the engine's asset loading phase. The static AssetCodecMapCodec field dictates the deserialization process from JSON files on disk.

- **Scope:** Once loaded, all CameraEffect instances are held in a static, in-memory AssetStore. Their lifetime is tied to the server or client session. They are effectively global, read-only data assets available for the entire application duration.

- **Destruction:** Objects are eligible for garbage collection only when the AssetStore is explicitly cleared or reloaded. This typically occurs on server shutdown or during a developer-initiated hot-reload of game assets.

## Internal State & Concurrency
- **State:** The internal state of a CameraEffect instance, such as its id, is populated from its source JSON file upon creation. After this initial loading, the object's state should be considered immutable. Modifying its state at runtime is an unsupported operation that can lead to unpredictable behavior across the engine.

- **Thread Safety:** Instances are inherently thread-safe for read operations due to their immutable nature post-initialization. However, the static `getAssetStore` method contains a potential race condition in its lazy initialization block.

    **WARNING:** The `if (ASSET_STORE == null)` check is not synchronized. Calling `getAssetStore` concurrently from multiple threads before it has been initialized can lead to the creation of multiple AssetStore instances, violating its singleton-like contract. Asset loading and retrieval must be performed in a thread-safe context, typically on the main game thread during the bootstrap sequence.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Returns the global, shared asset store for all CameraEffect assets. Subject to initialization race conditions. |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | Provides direct access to the underlying map of all loaded CameraEffect assets, keyed by their string ID. |
| getId() | String | O(1) | Returns the unique asset identifier for this effect (e.g., "hytale:shake_explosion_heavy"). |
| createCameraShakePacket() | abstract CameraShakeEffect | O(1) | Factory method to create a network packet that will trigger this effect on the client with default intensity. |
| createCameraShakePacket(float) | abstract CameraShakeEffect | O(1) | Factory method to create a network packet with a dynamically specified intensity multiplier. |

## Integration Patterns

### Standard Usage
The correct pattern is to retrieve a pre-loaded CameraEffect from the static AssetStore using its unique string identifier. This instance is then used to create a network packet for dispatch.

```java
// In a game system (e.g., handling an explosion event)
// 1. Retrieve the asset map.
IndexedLookupTableAssetMap<String, CameraEffect> effects = CameraEffect.getAssetMap();

// 2. Look up the desired effect by its ID.
CameraEffect explosionShake = effects.get("hytale:shake_explosion_heavy");

// 3. Create the network packet if the effect exists.
if (explosionShake != null) {
    CameraShakeEffect packet = explosionShake.createCameraShakePacket(1.5f);
    // networkSystem.sendToClients(packet);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create instances of CameraEffect subclasses with `new`. This bypasses the asset management system, resulting in an object that is not registered, not identifiable by ID, and will not be available to other game systems.

- **Concurrent Initialization:** Do not call `getAssetStore` or `getAssetMap` from multiple threads simultaneously during application startup. This can corrupt the static AssetStore reference. All asset initialization should be synchronized or confined to the main thread.

- **State Mutation:** Do not attempt to modify the fields of a CameraEffect object after it has been loaded. These assets are designed to be read-only data definitions.

## Data Pipeline
The flow for CameraEffect begins as a static data file and ends as a network command sent to a game client.

> Flow:
> JSON File on Disk -> AssetStore Loader -> **CameraEffect** (Deserialized Instance in Memory) -> Game Logic Lookup -> `createCameraShakePacket()` -> Network Packet -> Game Client Rendering Engine

