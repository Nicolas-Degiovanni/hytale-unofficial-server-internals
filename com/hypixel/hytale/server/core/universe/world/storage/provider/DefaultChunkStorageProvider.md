---
description: Architectural reference for DefaultChunkStorageProvider
---

# DefaultChunkStorageProvider

**Package:** com.hypixel.hytale.server.core.universe.world.storage.provider
**Type:** Singleton

## Definition
```java
// Signature
public class DefaultChunkStorageProvider implements IChunkStorageProvider {
```

## Architecture & Concepts
The DefaultChunkStorageProvider is not a storage implementation itself, but rather a **delegating strategy provider**. Its sole architectural purpose is to provide a stable, named entry point that redirects all chunk storage operations to the server's currently recommended concrete implementation.

This class embodies the **Strategy Pattern** at a configuration level. By default, it delegates all calls to an instance of IndexedStorageChunkStorageProvider. This provides a critical layer of indirection:
1.  **Configuration Stability:** Server and world configurations can reference the provider by its static ID, "Hytale", without being coupled to a specific storage engine.
2.  **Future-Proofing:** The Hytale engine team can change the underlying default storage mechanism in future updates (e.g., for performance optimization) by simply changing the static DEFAULT field. This change is transparent to existing worlds that are configured to use the "Hytale" provider.

It acts as a named alias, decoupling the logical concept of "default storage" from its physical implementation.

### Lifecycle & Ownership
-   **Creation:** The singleton INSTANCE is instantiated by the JVM during class loading. It is not created by any specific engine system; its lifecycle is static.
-   **Scope:** Application-scoped. The singleton instance persists for the entire lifetime of the server process.
-   **Destruction:** The object is destroyed only when the JVM shuts down. No manual cleanup is required or possible.

## Internal State & Concurrency
-   **State:** This class is **stateless and immutable**. Its only field of significance, DEFAULT, is a static final reference initialized at class-load time. It holds no per-request or per-world state.
-   **Thread Safety:** This class is inherently **thread-safe**. As a stateless singleton that performs simple delegation, its methods can be invoked concurrently from any thread without risk. All concerns regarding thread safety and I/O synchronization are deferred to the concrete IChunkLoader and IChunkSaver implementations it provides.

## API Surface
The public contract is defined by the IChunkStorageProvider interface. The methods act as simple pass-through calls.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getLoader(store) | IChunkLoader | O(1) | Delegates to the default provider to create a chunk loader. Throws IOException on underlying provider failure. |
| getSaver(store) | IChunkSaver | O(1) | Delegates to the default provider to create a chunk saver. Throws IOException on underlying provider failure. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in typical game logic. It is designed to be resolved from configuration by the world loading system via its registered CODEC and ID. A system would request a loader or saver from the provider that was configured for the world.

```java
// Hypothetical: World loader resolving the configured provider
// The "providerId" would be "Hytale" in a world.json file.
IChunkStorageProvider provider = StorageProviderRegistry.getProvider(providerId);

// The provider is then used to create the actual I/O handlers
IChunkLoader loader = provider.getLoader(world.getChunkStore());
IChunkSaver saver = provider.getSaver(world.getChunkStore());

// ... subsequent code uses the loader and saver, not the provider
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new DefaultChunkStorageProvider()`. It is a singleton and should only be accessed via its static INSTANCE field, though even that is rare outside of the codec system.
-   **Implementation Assumption:** Do not write code that assumes the delegated provider is of a specific type. Casting the result of `getLoader` or `getSaver` to a concrete class like `IndexedStorageChunkLoader` breaks the abstraction and will fail if the default provider is changed in a future engine version.

## Data Pipeline
This class is a **factory**, not a participant in the data stream. It is invoked once during world initialization to *create* the components that will later handle the data pipeline for chunk I/O.

> **Initialization Flow:**
> World Configuration File (references "Hytale" ID) -> Server Bootstrap -> Codec resolves ID to **DefaultChunkStorageProvider.INSTANCE** -> `getLoader()` / `getSaver()` called -> Returns concrete `IChunkLoader` / `IChunkSaver`
>
> **Resulting Data Flow (Chunks):**
> Disk / Database -> `IChunkLoader` -> Deserialization -> Game World

