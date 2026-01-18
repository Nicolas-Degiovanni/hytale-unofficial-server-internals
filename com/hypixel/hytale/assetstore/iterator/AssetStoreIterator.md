---
description: Architectural reference for AssetStoreIterator
---

# AssetStoreIterator

**Package:** com.hypixel.hytale.assetstore.iterator
**Type:** Transient Utility

## Definition
```java
// Signature
public class AssetStoreIterator implements Iterator<AssetStore<?, ?, ?>>, Closeable {
```

## Architecture & Concepts
The AssetStoreIterator is a specialized iterator designed to solve the critical problem of asset dependency ordering during the engine's loading phase. Unlike a standard Java Iterator which traverses a collection in a fixed sequence, this class implements a form of topological sorting. Its primary function is to yield AssetStore instances only when their declared dependencies have already been processed.

This component is central to the asset system's stability. It ensures that foundational assets (like textures and materials) are fully loaded and registered before dependent assets (like models or entity definitions) attempt to reference them. The iterator maintains an internal list of pending AssetStores, and on each call to *next*, it scans this list for a store whose dependencies are no longer pending.

An important, non-standard behavior is that the *next* method can return null even when *hasNext* is true. This indicates that all remaining AssetStores are currently blocked, waiting on dependencies that are also in the pending list. The consuming system must be designed to handle this state, which could signify a temporary stall or a permanent circular dependency.

## Lifecycle & Ownership
-   **Creation:** Instantiated directly via its constructor, typically by a high-level asset management service at the beginning of a loading sequence. It is initialized with a collection of all AssetStore objects that need to be processed.
-   **Scope:** The iterator is short-lived. Its existence is scoped to a single, complete asset loading operation. It holds mutable state representing the remaining work and is discarded once the iteration is complete.
-   **Destruction:** The object becomes eligible for garbage collection as soon as the reference to it is dropped, which is typically after the loading loop terminates. The implementation of the Closeable interface, with an empty *close* method, suggests a design pattern encouraging its use within a try-with-resources block for deterministic cleanup, even if no explicit resource release is currently needed.

## Internal State & Concurrency
-   **State:** The AssetStoreIterator is highly stateful and mutable. Its core state is the internal *list* of pending AssetStores. Each successful call to the *next* method removes an element from this list, shrinking its state over time.
-   **Thread Safety:** This class is **not thread-safe**. Its internal state, an ArrayList, is accessed and modified without any synchronization. Concurrent access to its methods from multiple threads will result in unpredictable behavior, including ConcurrentModificationException. It is designed to be used exclusively by a single thread that orchestrates the asset loading process.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hasNext() | boolean | O(1) | Returns true if there are one or more AssetStores remaining to be processed. |
| next() | AssetStore | O(N*M) | Returns a dependency-free AssetStore or null if all remaining stores are blocked. N is the number of remaining stores; M is the average number of dependencies per store. |
| size() | int | O(1) | Returns the current number of unprocessed AssetStores. |
| isWaitingForDependencies(store) | boolean | O(M) | Checks if the given store has dependencies that are still in the internal pending list. M is the number of dependencies for the store. |
| isBeingWaitedFor(store) | boolean | O(N*D) | Checks if any other pending store lists the given store as a dependency. N is the number of remaining stores; D is the average number of dependencies per store. |

## Integration Patterns

### Standard Usage
The correct usage pattern involves a loop that can handle the null return from the *next* method. The calling code is responsible for detecting a potential deadlock if *next* repeatedly returns null while *hasNext* remains true.

```java
// Obtain all stores that need to be loaded
Collection<AssetStore<?, ?, ?>> allStores = AssetRegistry.getAssetStores();
AssetStoreIterator iterator = new AssetStoreIterator(allStores);

int deadlockCounter = 0;
int maxAttempts = allStores.size() * allStores.size(); // A safe upper bound

while (iterator.hasNext()) {
    AssetStore<?, ?, ?> store = iterator.next();

    if (store != null) {
        // A dependency-free store is available, process it
        loadAssetsFor(store);
        deadlockCounter = 0; // Reset counter on progress
    } else {
        // No store is ready, indicates a stall or cycle
        deadlockCounter++;
        if (deadlockCounter > maxAttempts) {
            throw new IllegalStateException("Asset dependency deadlock detected!");
        }
        // Optional: Thread.sleep(1) to prevent a tight spin-lock
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Assuming Non-Null Return:** Never assume *next* will return a valid object if *hasNext* is true. This violates the standard Iterator contract and is a primary source of NullPointerExceptions if not handled.
-   **Concurrent Access:** Do not share an instance of AssetStoreIterator across multiple threads. The internal state modifications are not synchronized and will corrupt the iteration.
-   **Ignoring Deadlocks:** Failing to implement a check for circular dependencies can cause the loading process to enter an infinite loop, effectively hanging the application.

## Data Pipeline
The AssetStoreIterator does not transform data itself but rather orchestrates the order of processing for other systems. It acts as a gatekeeper in the asset loading pipeline.

> Flow:
> AssetRegistry -> Collection of all AssetStores -> **AssetStoreIterator** -> Yields one dependency-safe AssetStore -> Asset Loading Service -> Assets loaded to memory

