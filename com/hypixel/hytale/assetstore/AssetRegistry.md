---
description: Architectural reference for AssetRegistry
---

# AssetRegistry

**Package:** com.hypixel.hytale.assetstore
**Type:** Static Utility / Singleton Registry

## Definition
```java
// Signature
public class AssetRegistry {
```

## Architecture & Concepts
The AssetRegistry is a static, globally-accessible service that functions as the central directory for all **AssetStore** instances within the engine. It provides a decoupled, type-safe mechanism for different game systems to register and retrieve specialized asset managers.

Its core responsibilities are twofold:

1.  **Store Management:** It maintains a map linking an asset's class type (e.g., **Texture.class**) to the singleton **AssetStore** responsible for managing assets of that type. This pattern allows any system to request the correct manager for a given asset type without needing a direct dependency on the manager's implementation.

2.  **Tag Interning:** It implements a high-performance tag interning system. String-based tags, which are common for identifying game objects and properties, are mapped to unique integer identifiers. This is a critical optimization that allows for faster comparisons and lookups (integer vs. string) and reduces network payload size by serializing compact integer IDs instead of full strings.

The registry also integrates with the engine's event system, dispatching **RegisterAssetStoreEvent** and **RemoveAssetStoreEvent** to notify other systems of changes to the available asset stores.

## Lifecycle & Ownership
-   **Creation:** As a static class, the AssetRegistry is not instantiated. Its static fields are initialized by the JVM ClassLoader when the class is first referenced. The static initializer block configures the default return values for its internal maps.
-   **Scope:** The AssetRegistry is global and its state persists for the entire lifetime of the application. It is a foundational service that is available immediately after class loading.
-   **Destruction:** The registry and its contents are destroyed only when the Java Virtual Machine shuts down. There is no manual cleanup or teardown process.

## Internal State & Concurrency
-   **State:** The internal state of the AssetRegistry is highly mutable. It contains several static maps (**storeMap**, **TAG_MAP**, **CLIENT_TAG_MAP**) that are populated and modified at runtime as the application initializes and runs.

-   **Thread Safety:** This class is explicitly designed to be thread-safe and is safe for concurrent access from multiple threads.
    -   **Store Operations:** All modifications to the **storeMap** via the register and unregister methods are protected by a **ReentrantReadWriteLock** (**ASSET_LOCK**). This ensures atomic updates while allowing concurrent reads.
    -   **Tag Operations:** All access to the tag maps is guarded by a **StampedLock** (**TAG_LOCK**), a high-performance lock suitable for read-heavy workloads.
    -   **Tag ID Generation:** New tag indices are generated using an **AtomicInteger**, which provides a lock-free, thread-safe mechanism for creating unique IDs.

    **WARNING:** While the methods are thread-safe, callers are responsible for ensuring logical consistency. For example, retrieving an **AssetStore** and then operating on it is not a single atomic operation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(assetStore) | S | O(1) | Registers an AssetStore instance. Acquires a write lock. Throws IllegalArgumentException if a store for the asset type already exists. Dispatches a RegisterAssetStoreEvent. |
| unregister(assetStore) | void | O(1) | Removes an AssetStore instance. Acquires a write lock. Dispatches a RemoveAssetStoreEvent. |
| getAssetStore(tClass) | AssetStore | O(1) | Retrieves the registered AssetStore for a given asset class. This operation is non-locking but relies on the thread-safety of the underlying concurrent map. |
| getOrCreateTagIndex(tag) | int | O(1) | Retrieves the integer ID for a string tag. If the tag does not exist, it is created atomically. Acquires a write lock on the tag map. |
| getTagIndex(tag) | int | O(1) | Retrieves the integer ID for a string tag. Returns TAG_NOT_FOUND if it does not exist. Acquires a read lock on the tag map. |

## Integration Patterns

### Standard Usage
Systems should register their respective **AssetStore** instances during the engine's initialization phase. Other systems can then retrieve these stores on-demand to access assets.

```java
// During engine or mod initialization
ModelAssetStore modelStore = new ModelAssetStore(engine.getEventBus());
AssetRegistry.register(modelStore);

// In another system that needs to access models
AssetStore<ModelKey, Model, ModelMap> store = AssetRegistry.getAssetStore(Model.class);
Model playerModel = store.get(new ModelKey("player_character"));
```

### Anti-Patterns (Do NOT do this)
-   **Late Registration:** Do not register an **AssetStore** after the main loading phase is complete. Other systems may have already tried and failed to retrieve it, resulting in initialization errors or unpredictable behavior.
-   **Holding References to the Unmodifiable Map:** The map returned by **getStoreMap** is an unmodifiable *view* of the internal map. Do not cache this map and assume its contents are static, as stores can be unregistered at runtime. Always query the registry for the most current state.
-   **Forgetting to Unregister:** In dynamic environments (e.g., mod unloading, server state changes), failing to call **unregister** will result in a memory leak, as the **AssetRegistry** will hold a permanent reference to the stale **AssetStore**.

## Data Pipeline
The AssetRegistry acts as a central directory, not a data processor. Its primary flows involve registration and lookup.

**AssetStore Registration Flow:**
> System Initializer -> `new MyAssetStore()` -> `AssetRegistry.register(store)` -> **AssetRegistry** (Acquires lock, updates internal map) -> Event Bus -> `RegisterAssetStoreEvent` -> Listening Systems

**Tag Creation & Lookup Flow:**
> Game Logic -> `AssetRegistry.getOrCreateTagIndex("hytale:stone_sword")` -> **AssetRegistry** (Acquires lock, checks map, creates new ID if needed) -> Returns Integer ID -> Game Logic (uses ID for networking, components, etc.)

