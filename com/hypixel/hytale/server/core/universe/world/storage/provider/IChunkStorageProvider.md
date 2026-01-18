---
description: Architectural reference for IChunkStorageProvider
---

# IChunkStorageProvider

**Package:** com.hypixel.hytale.server.core.universe.world.storage.provider
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface IChunkStorageProvider {
```

## Architecture & Concepts
The **IChunkStorageProvider** interface is a critical abstraction within the server's world persistence layer. It embodies the Strategy design pattern, decoupling the core world engine from the underlying physical storage medium. Its sole responsibility is to act as a factory for concrete **IChunkLoader** and **IChunkSaver** implementations.

This abstraction allows the server to support various storage backends—such as a local file system, a remote database, or even an in-memory store for testing—without altering the high-level chunk management logic. Implementations of this interface represent a specific storage strategy (e.g., Anvil format, RocksDB, etc.).

The static **CODEC** field is a significant architectural feature. It indicates that the choice of storage provider is a configurable aspect of a world, likely defined in a server or world configuration file. The engine uses this codec to deserialize the configuration and instantiate the correct provider implementation at runtime.

## Lifecycle & Ownership
As an interface, **IChunkStorageProvider** itself has no lifecycle. The following pertains to its concrete implementations.

-   **Creation:** An implementation is instantiated by the world loading mechanism, typically by deserializing world configuration data using the static **CODEC**. This happens once when a world is initialized.
-   **Scope:** The provider instance is scoped to a single world session. It persists for the entire duration that its associated world is loaded in memory.
-   **Destruction:** The instance is eligible for garbage collection when the world is unloaded and all references to it, and the loader/saver it created, are released.

## Internal State & Concurrency
-   **State:** This interface is stateless by definition. However, concrete implementations are expected to be stateful. They may hold configuration details, file paths, or database connection pools required to create functional loaders and savers.
-   **Thread Safety:** The interface contract does not enforce thread safety. Implementations are not expected to be thread-safe, as the factory methods `getLoader` and `getSaver` are typically invoked synchronously during the world's initialization phase.

    **WARNING:** The **IChunkLoader** and **IChunkSaver** instances returned by this provider *must* be designed for concurrent access. Chunk I/O is a highly parallelized operation, and these worker objects will be accessed from multiple world-gen and game-tick threads.

## API Surface
The public contract is minimal, focused entirely on factory duties.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodecMapCodec | O(1) | Static codec for serializing and deserializing provider configurations. |
| getLoader(Store) | IChunkLoader | O(N) | Creates and returns a chunk loader for the specified store. May throw IOException on I/O or configuration errors. Complexity can be high due to initial I/O operations (e.g., opening files, establishing connections). |
| getSaver(Store) | IChunkSaver | O(N) | Creates and returns a chunk saver for the specified store. May throw IOException on I/O or configuration errors. Complexity is similar to getLoader. |

## Integration Patterns

### Standard Usage
The provider is retrieved from world configuration and used to bootstrap the persistence workers for that world. This is a one-time setup operation per world load.

```java
// Pseudo-code for world initialization
IChunkStorageProvider storageProvider = worldConfig.getStorageProvider();
Store<ChunkStore> chunkStore = this.world.getChunkStore();

// Create the workers that will live for the world's session
IChunkLoader loader = storageProvider.getLoader(chunkStore);
IChunkSaver saver = storageProvider.getSaver(chunkStore);

// Register workers with the world's storage manager
this.world.getStorageManager().setWorkers(loader, saver);
```

### Anti-Patterns (Do NOT do this)
-   **Repeated Invocations:** Do not call `getLoader` or `getSaver` multiple times for the same world. These methods are not idempotent and may be expensive, potentially creating new file handles or database connections on each call. They are intended to be called only once during initialization.
-   **Ignoring Exceptions:** An `IOException` from these methods indicates a fatal misconfiguration or a problem with the storage medium. It must be handled by aborting the world load.
-   **Sharing Workers:** The loader and saver instances returned are tied to the specific `Store<ChunkStore>` passed as an argument. Do not attempt to share these worker instances across different worlds or chunk stores.

## Data Pipeline
This interface acts as the entry point for configuring the world's data persistence pipeline.

> Flow:
> World Configuration File -> **CODEC** Deserialization -> **IChunkStorageProvider Instance** -> `getLoader()` / `getSaver()` -> I/O Worker Objects -> World Storage Engine

