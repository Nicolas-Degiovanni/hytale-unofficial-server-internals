---
description: Architectural reference for OggVorbisInfoCache
---

# OggVorbisInfoCache

**Package:** com.hypixel.hytale.server.core.asset.common
**Type:** Utility

## Definition
```java
// Signature
public class OggVorbisInfoCache {
```

## Architecture & Concepts

The OggVorbisInfoCache is a specialized, low-level utility designed to optimize audio metadata retrieval. Its primary function is to parse Ogg Vorbis audio files on-demand to extract key information—specifically channel count, sample rate, and duration—and cache these results in memory.

This class serves as a performance-critical bridge between the asset system and the audio engine. By caching this metadata, the system avoids the expensive I/O and byte-level parsing operations that would otherwise be required every time a sound effect or music track is accessed. The cache is keyed by the asset's unique string name, ensuring that metadata for any given audio file is computed only once per application session, unless explicitly invalidated.

The core parsing logic operates directly on the raw byte buffer of an asset. It performs a highly specific byte-pattern search to locate the Vorbis identification header and the final Ogg page to calculate the total duration. This direct manipulation of binary data makes the component extremely fast but also tightly coupled to the Ogg Vorbis format specification.

## Lifecycle & Ownership

-   **Creation:** The OggVorbisInfoCache is a static utility class and is never instantiated. Its internal state, a ConcurrentHashMap, is initialized statically when the class is first loaded by the Java Virtual Machine.
-   **Scope:** The cache and its contained data persist for the entire lifetime of the application. Entries are added on-demand as new audio assets are accessed.
-   **Destruction:** There is no global destruction or cleanup mechanism. Individual cache entries must be explicitly removed by calling the invalidate method. This is a critical responsibility of the asset management system, which must call invalidate whenever an Ogg Vorbis asset is updated or unloaded to prevent stale data.

## Internal State & Concurrency

-   **State:** The class maintains a single, static, mutable state container: the vorbisFiles map. This ConcurrentHashMap stores immutable OggVorbisInfo objects. Once an OggVorbisInfo object is created and cached, its internal values (channels, sampleRate, duration) cannot be changed.
-   **Thread Safety:** This class is thread-safe and designed for high-concurrency environments.
    -   The use of ConcurrentHashMap ensures that lookups, insertions, and removals on the cache do not require external synchronization.
    -   The primary get methods return a CompletableFuture, allowing the expensive file I/O and parsing operations to occur asynchronously without blocking the calling thread.
    -   **WARNING:** The getNow methods are synchronous and will block the calling thread until the operation is complete. Calling getNow for an uncached asset from a performance-sensitive thread (such as the main server tick loop) will cause significant stalls and must be avoided.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(String name) | CompletableFuture | O(N) | Asynchronously retrieves metadata for an asset by name. Parses and caches the asset if not present. N is the size of the asset file. |
| get(CommonAsset asset) | CompletableFuture | O(N) | Asynchronously retrieves metadata for a given CommonAsset instance. Parses and caches the asset if not present. |
| getNow(String name) | OggVorbisInfo | O(N) | **Blocking.** Synchronously retrieves metadata. Throws unchecked exceptions on failure. **Use with extreme caution.** |
| getNow(CommonAsset asset) | OggVorbisInfo | O(N) | **Blocking.** Synchronously retrieves metadata for a given asset. **Use with extreme caution.** |
| invalidate(String name) | void | O(1) | Removes a specific asset's metadata from the cache. Essential for asset hot-reloading. |

## Integration Patterns

### Standard Usage

The intended usage is to asynchronously request the metadata and chain subsequent logic to the resulting CompletableFuture. This prevents blocking critical engine threads.

```java
// Asynchronously retrieve info and schedule work on the audio engine
CompletableFuture<OggVorbisInfo> futureInfo = OggVorbisInfoCache.get("sounds/music/overworld_theme.ogg");

futureInfo.thenAccept(info -> {
    if (info != null) {
        // Safely use the metadata on the appropriate thread
        audioEngine.prepareStream(info);
    }
});
```

### Anti-Patterns (Do NOT do this)

-   **Blocking Critical Threads:** Never call getNow from the main server thread or rendering thread. The potential for file I/O and parsing can freeze the application. Always prefer the asynchronous get method.
-   **Stale Cache Data:** Do not modify or replace Ogg Vorbis asset files without calling invalidate on the corresponding asset name. Failure to do so will result in the audio engine operating with incorrect metadata (e.g., wrong duration, channel count).

## Data Pipeline

The flow of data for a cache miss is a multi-stage process involving the core asset system.

> Flow:
> Asset Name -> CommonAssetRegistry -> CommonAsset -> Raw Byte Array -> **OggVorbisInfoCache** (Parse & Cache) -> OggVorbisInfo -> Audio Engine

