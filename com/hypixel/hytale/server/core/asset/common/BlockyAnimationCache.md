---
description: Architectural reference for BlockyAnimationCache
---

# BlockyAnimationCache

**Package:** com.hypixel.hytale.server.core.asset.common
**Type:** Utility

## Definition
```java
// Signature
public class BlockyAnimationCache {
```

## Architecture & Concepts

The BlockyAnimationCache is a global, in-memory, on-demand caching layer for deserialized blocky animation data. It serves as a high-performance bridge between the asset system and game logic that requires animation metadata. Its primary architectural function is to abstract and accelerate the costly process of file I/O and JSON parsing, ensuring that frequently accessed animation data is readily available in its deserialized object form.

This class operates as a static utility, exposing a global cache that is shared across the entire server process. It employs a lazy-loading strategy: animation data is only loaded from the underlying asset system and parsed upon its first request. Subsequent requests for the same animation are served directly from the in-memory cache, providing a significant performance boost.

The API is designed with concurrency in mind, offering asynchronous, non-blocking methods that return a CompletableFuture. This allows calling systems, such as the entity component system or animation controllers, to request animation data without stalling the main game loop.

## Lifecycle & Ownership

-   **Creation:** The BlockyAnimationCache is a static utility class and is never instantiated. Its internal state, a static ConcurrentHashMap, is initialized by the JVM during class loading.
-   **Scope:** The cache and its contents are global and persist for the entire lifetime of the server application. It is a long-lived, session-scoped resource.
-   **Destruction:** The cache is only destroyed upon JVM shutdown. Individual entries can be evicted programmatically by calling the invalidate method, but there is no automatic garbage collection or eviction policy (e.g., LRU, LFU).

## Internal State & Concurrency

-   **State:** The core state is a single static ConcurrentHashMap named animations. This map is mutable, storing string-based asset names as keys and the corresponding BlockyAnimation objects as values. The cache grows dynamically as new animations are requested.

-   **Thread Safety:** This class is designed to be thread-safe for concurrent read and write operations.
    -   The use of ConcurrentHashMap guarantees atomic reads, writes, and removals from the cache map itself.
    -   The primary get methods are non-blocking and return a CompletableFuture, delegating I/O and parsing work to a separate thread pool. This makes them safe to call from any thread, including performance-critical ones.
    -   **Warning:** A subtle race condition exists. If two threads request the same *uncached* asset simultaneously, both may miss the cache check, proceed to load and decode the asset from disk, and then attempt to write to the cache. While the final write is atomic, the redundant I/O and CPU work is inefficient. This is a classic "cache stampede" scenario on a per-asset basis.

## API Surface

| Symbol                                               | Type                                                        | Complexity     | Description                                                                                                                              |
| :--------------------------------------------------- | :---------------------------------------------------------- | :------------- | :--------------------------------------------------------------------------------------------------------------------------------------- |
| get(String name)                                     | CompletableFuture<BlockyAnimation>                          | O(1) / O(N)    | Asynchronously retrieves an animation. O(1) on cache hit. O(N) on miss, where N is asset size, as it triggers I/O and parsing.           |
| get(CommonAsset asset)                               | CompletableFuture<BlockyAnimation>                          | O(1) / O(N)    | Asynchronously retrieves an animation using an asset handle. Behavior is identical to the string-based variant.                         |
| getNow(String name)                                  | BlockyAnimation                                             | O(1) / O(N)    | **Blocking.** Synchronously retrieves an animation, blocking the calling thread on a cache miss until I/O and parsing are complete.      |
| getNow(CommonAsset asset)                            | BlockyAnimation                                             | O(1) / O(N)    | **Blocking.** Synchronously retrieves an animation using an asset handle. Identical blocking behavior to the string-based variant. |
| invalidate(String name)                              | void                                                        | O(1)           | Removes a specific animation from the cache, forcing a reload on the next request.                                                       |

## Integration Patterns

### Standard Usage

The preferred integration pattern is to use the asynchronous get method and chain subsequent logic using the CompletableFuture API. This avoids blocking critical threads.

```java
// Asynchronously request an animation and apply it when loaded
CompletableFuture<BlockyAnimation> futureAnim = BlockyAnimationCache.get("hytale:player_walk");

futureAnim.thenAccept(animation -> {
    if (animation != null) {
        // Logic to apply the animation data to a model or entity
        playerEntity.getAnimationController().play(animation);
    } else {
        BlockyAnimationCache.LOGGER.warn("Failed to load animation: hytale:player_walk");
    }
});
```

### Anti-Patterns (Do NOT do this)

-   **Blocking Critical Threads:** Never call getNow or .join() on a CompletableFuture from a performance-sensitive context like the main server tick loop. This will cause the entire thread to stall on file I/O, leading to severe performance degradation and server lag.

    ```java
    // BAD: This will freeze the thread if the animation is not already cached.
    BlockyAnimation anim = BlockyAnimationCache.getNow("hytale:player_attack");
    ```

-   **Excessive Invalidation:** Do not call invalidate frequently or as part of a normal game logic loop. The purpose of the cache is to avoid I/O. Constant invalidation negates this benefit and can cause performance issues equivalent to not having a cache at all. Invalidation should only be used in response to asset hot-reloading or specific administrative actions.

## Data Pipeline

The data pipeline for a cache miss is a multi-stage process that transforms a raw asset file into a usable in-memory object.

> Flow:
> 1.  **Asset Request:** A game system calls `BlockyAnimationCache.get(name)`.
> 2.  **Cache Miss:** The internal map does not contain the requested name.
> 3.  **Registry Lookup:** The cache queries the `CommonAssetRegistry` to resolve the name into a `CommonAsset` handle.
> 4.  **Blob Fetch:** The `CommonAsset.getBlob()` method is called, which asynchronously reads the raw animation file (a JSON document) from disk into a byte array.
> 5.  **Decoding:** The byte array is converted to a UTF-8 string.
> 6.  **JSON Parsing:** A `RawJsonReader` and the static `BlockyAnimation.CODEC` are used to deserialize the JSON string into a `BlockyAnimation` Java object.
> 7.  **Cache Population:** The newly created `BlockyAnimation` object is inserted into the static `animations` ConcurrentHashMap.
> 8.  **Future Completion:** The `CompletableFuture` returned in step 1 is completed with the `BlockyAnimation` object, triggering any dependent logic.

