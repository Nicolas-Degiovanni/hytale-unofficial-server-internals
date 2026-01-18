---
description: Architectural reference for AudioCategory
---

# AudioCategory

**Package:** com.hypixel.hytale.server.core.asset.type.audiocategory.config
**Type:** Data Asset

## Definition
```java
// Signature
public class AudioCategory
   implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, AudioCategory>>,
   NetworkSerializable<com.hypixel.hytale.protocol.AudioCategory> {
```

## Architecture & Concepts
The AudioCategory class is a data-driven asset that defines a hierarchical grouping for sound events. It is a fundamental component of the audio engine's configuration system, allowing for centralized volume control over entire classes of sounds (e.g., music, effects, ambient).

Architecturally, this class represents a node in a tree of audio settings. Each AudioCategory can inherit from a parent, creating a cascading volume system analogous to an audio bus in digital audio workstations. The volume of a child category is multiplied with the volume of its parent, allowing for fine-grained control. For instance, a *footsteps* category might be a child of an *effects* category, which is a child of the *master* category. Adjusting the *effects* volume will proportionally affect *footsteps* and all other child sounds.

This class is designed to be loaded from JSON asset files by the engine's AssetStore. Its primary roles are:
1.  To model the configuration data from an asset file in memory.
2.  To resolve the final, inherited volume by traversing its parent hierarchy.
3.  To serialize its state into a network-optimized packet for client-server synchronization.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the AssetStore during the asset loading phase. The static CODEC field defines the deserialization logic, which reads JSON data, instantiates an AudioCategory object, and populates its fields, including resolving its parent relationship. Manual instantiation is an anti-pattern and will result in a misconfigured, non-functional object.
-   **Scope:** An AudioCategory object has a global, session-wide scope. Once loaded into the static AssetStore, it persists for the entire lifetime of the server or client process.
-   **Destruction:** Objects are eligible for garbage collection only when the AssetStore is cleared or the application terminates.

## Internal State & Concurrency
-   **State:** The class holds mutable state, primarily the *id* and *volume*. This state is set once during asset deserialization and should be treated as immutable thereafter. A critical piece of internal state is the *cachedPacket*, a SoftReference to the network-serialized version of this object. This cache significantly reduces the cost of repeatedly calculating the inherited volume for network transmission. The volume is stored internally as a linear gain value, converted from the more user-friendly decibel format specified in the asset files.

-   **Thread Safety:** This class is **not thread-safe**. Its fields are accessed without any synchronization mechanisms. All mutation is expected to occur on a single thread during the initial asset loading phase. The static `getAssetStore` method contains a potential race condition in its lazy initialization block, which is only safe if the first call is guaranteed to happen in a single-threaded context, such as application startup.

    **WARNING:** Concurrent access to an AudioCategory instance, especially calls to `toPacket` from multiple threads, can lead to race conditions in the creation of the cached packet. All interactions with the AssetStore and its managed assets must be synchronized externally or confined to a single thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Returns the global, static AssetStore for all AudioCategory assets. |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | Returns the underlying map holding all loaded AudioCategory assets. |
| getId() | String | O(1) | Returns the unique identifier for this category. |
| getVolume() | float | O(1) | Returns the local, non-inherited volume as a linear gain value. |
| toPacket() | com.hypixel.hytale.protocol.AudioCategory | O(N) | Converts the asset into a network packet. Traverses the parent hierarchy (N deep) to calculate the final, combined volume. Results are cached via a SoftReference. |

## Integration Patterns

### Standard Usage
The primary interaction pattern is to retrieve a pre-loaded AudioCategory from the central AssetStore via its unique string identifier. This object is then typically used to configure audio systems or to be sent to a client.

```java
// How a developer should normally use this
// Retrieve the central store for audio categories
AssetStore<String, AudioCategory, ?> store = AudioCategory.getAssetStore();

// Get a specific category by its asset key (e.g., "hytale:music")
AudioCategory musicCategory = store.getAssetMap().getAsset("hytale:music");

if (musicCategory != null) {
    // Convert to a network packet to send to a client
    com.hypixel.hytale.protocol.AudioCategory packet = musicCategory.toPacket();
    // networkSubsystem.send(packet);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new AudioCategory()`. This bypasses the AssetStore and the CODEC, resulting in an object that is not part of the inheritance hierarchy and lacks critical data.
-   **State Mutation:** Do not modify the state of an AudioCategory object after it has been loaded. The asset system is the single source of truth, and runtime modifications will lead to inconsistent behavior and will be overwritten on an asset reload.
-   **Ignoring Hierarchy:** Reading only the local `getVolume()` value is often incorrect. The effective volume must be calculated by traversing the parent chain, which is handled automatically by the `toPacket()` method.

## Data Pipeline
The data flow for an AudioCategory begins as a static file and ends as a network packet delivered to a game client.

> Flow:
> JSON Asset File -> AssetStore Loader -> **AudioCategory.CODEC** -> In-Memory **AudioCategory** Instance -> `toPacket()` call -> Protocol Buffer Packet -> Network Layer -> Game Client

