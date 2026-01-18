---
description: Architectural reference for PrefabBufferUtil
---

# PrefabBufferUtil

**Package:** com.hypixel.hytale.server.core.prefab.selection.buffer
**Type:** Utility

## Definition
```java
// Signature
public class PrefabBufferUtil {
```

## Architecture & Concepts

The PrefabBufferUtil class is a static utility that serves as the primary access point for loading and caching game prefabs. It is a critical performance component, designed to abstract the underlying prefab storage format and minimize disk I/O and deserialization overhead.

The core architectural concept is a **two-tier caching system**:

1.  **In-Memory Cache (Tier 1):** A static, concurrent map holds weak references to recently accessed PrefabBuffer instances. This provides the fastest possible access for frequently used prefabs, avoiding all disk I/O. The use of WeakReference ensures that unused prefabs do not cause memory leaks and can be garbage collected under memory pressure.

2.  **On-Disk Cache (Tier 2):** The system transparently manages two file formats for prefabs: a human-readable JSON format (.prefab.json) and a highly optimized, binary format (.lpf). When a prefab is requested for the first time from a JSON source, PrefabBufferUtil deserializes it, then serializes it into the binary LPF format. This binary version is stored in a dedicated cache directory.

Subsequent requests for the same prefab will load the pre-compiled LPF version directly, bypassing the expensive JSON parsing and BSON conversion steps entirely. The system validates cache freshness by comparing file modification times, ensuring that changes to source JSON or asset packs correctly invalidate the on-disk cache.

This utility acts as a gatekeeper for all prefab data, ensuring that game systems receive prefab data in a consistent and highly efficient manner.

## Lifecycle & Ownership

-   **Creation:** As a static utility class, PrefabBufferUtil is never instantiated. Its static state, including the in-memory CACHE map, is initialized by the JVM ClassLoader when the class is first referenced.
-   **Scope:** The static state and the on-disk cache persist for the entire lifetime of the server application. The on-disk cache in the .cache/prefabs directory persists even between server restarts.
-   **Destruction:** The in-memory cache is cleared when the application terminates. Individual cached entries may be garbage collected earlier if they are no longer strongly referenced elsewhere in the application, due to the WeakReference mechanism.

## Internal State & Concurrency

-   **State:** The primary internal state is the static CACHE map, which is mutable. This map stores weak references to CachedEntry objects, each of which contains a PrefabBuffer and a StampedLock. The state is designed to be volatile and performance-oriented.

-   **Thread Safety:** This class is **thread-safe** and designed for high-concurrency environments.
    -   The top-level CACHE is a ConcurrentHashMap, providing safe atomic operations.
    -   The loading logic within the getCached method is protected by a sophisticated locking strategy. A StampedLock is used on a per-entry basis to allow multiple threads to read a fully loaded prefab concurrently. When a prefab needs to be loaded from disk, the lock is escalated to an exclusive write lock, ensuring that the I/O operation happens only once. This minimizes contention and avoids redundant work.
    -   I/O operations can be performed asynchronously via methods that return a CompletableFuture, preventing the calling thread from blocking on disk access.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCached(Path path) | IPrefabBuffer | O(1) to O(N) | The primary entry point. Retrieves a prefab buffer, transparently utilizing the in-memory and on-disk caches. Complexity is O(1) for an in-memory hit, but O(N) for a full disk read and deserialization, where N is the size of the prefab. |
| writeToFileAsync(prefab, path) | CompletableFuture<Void> | O(N) | Asynchronously serializes and writes a PrefabBuffer to a specified path in the binary LPF format. |
| readFromFileAsync(path) | CompletableFuture<PrefabBuffer> | O(N) | Asynchronously reads and deserializes a PrefabBuffer from a binary LPF file at the specified path. |
| readFromFile(path) | PrefabBuffer | O(N) | Synchronous, blocking version of readFromFileAsync. **WARNING:** This will block the calling thread on disk I/O. |

## Integration Patterns

### Standard Usage

The getCached method should be considered the sole entry point for retrieving prefab data. All other game systems should use this method to ensure they benefit from the caching layers.

```java
// Correctly retrieve a prefab using the caching system
Path prefabPath = Path.of("prefabs/structures/example_house.prefab.json");
IPrefabBuffer buffer = PrefabBufferUtil.getCached(prefabPath);

// Use the buffer...
```

### Anti-Patterns (Do NOT do this)

-   **Bypassing the Cache:** Do not call methods like loadFromJson or loadFromLPF directly. These are internal helpers. Bypassing getCached will defeat the in-memory cache and its concurrency controls, leading to severe performance degradation and potential race conditions.
-   **Using Synchronous I/O on Hot Paths:** Avoid calling the synchronous readFromFile method on performance-critical threads, such as the main game loop. Prefer the asynchronous variants like readFromFileAsync to prevent blocking and server stalls.
-   **Ignoring Errors:** The loading methods can throw an Error on critical failures, such as a missing file. This indicates a fatal configuration or asset issue and must be handled appropriately by the calling system, as it is more severe than a standard Exception.

## Data Pipeline

The data flow for a prefab request is determined by the state of the caches.

> **Flow: In-Memory Cache Hit (Fastest)**
> Path Request -> **PrefabBufferUtil.getCached** -> ConcurrentHashMap Lookup -> Return existing PrefabBuffer

> **Flow: On-Disk Cache Hit**
> Path Request -> **PrefabBufferUtil.getCached** -> In-Memory Miss -> Check for .lpf file -> **loadFromLPF** -> Binary Deserialization -> Populate In-Memory Cache -> Return PrefabBuffer

> **Flow: Cache Miss (Slowest)**
> Path Request -> **PrefabBufferUtil.getCached** -> In-Memory Miss -> .lpf Miss -> **loadFromJson** -> JSON/BSON Deserialization -> Create PrefabBuffer -> Populate In-Memory Cache -> (Async) **writeToFileAsync** -> Write .lpf to disk -> Return PrefabBuffer

