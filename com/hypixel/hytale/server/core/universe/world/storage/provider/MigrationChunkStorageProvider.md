---
description: Architectural reference for MigrationChunkStorageProvider
---

# MigrationChunkStorageProvider

**Package:** com.hypixel.hytale.server.core.universe.world.storage.provider
**Type:** Provider / Factory

## Definition
```java
// Signature
public class MigrationChunkStorageProvider implements IChunkStorageProvider {
```

## Architecture & Concepts
The MigrationChunkStorageProvider is a strategic component within the world storage subsystem, designed to facilitate the seamless, on-the-fly migration of chunk data from legacy formats to a canonical format. It functions as a specialized factory that produces chunk loader and saver instances with distinct, asymmetrical behaviors.

This provider implements a **Composite** and **Chain of Responsibility** pattern for loading operations, and a **Proxy** pattern for saving operations.

-   **Loading (Read Path):** When a chunk needs to be loaded, the provider generates a MigrationChunkLoader. This loader is configured with an ordered list of subordinate IChunkStorageProvider instances. It attempts to load a requested chunk from the first provider in the list. If the load fails (e.g., the chunk does not exist in that format), it proceeds to the next provider in the chain until the chunk is found or all providers are exhausted. This allows the server to read from multiple potential data sources, such as an old region file format and a new database format, transparently.

-   **Saving (Write Path):** For saving operations, the provider is configured with a single, authoritative IChunkStorageProvider. All write operations are unconditionally proxied to this destination provider.

This architectural separation of read and write strategies is the key to its function. As the game engine loads chunks from any of the old formats, any subsequent modifications and saves will write them *only* to the new, designated format. Over time, the world's data is organically migrated without requiring an offline conversion utility.

## Lifecycle & Ownership
-   **Creation:** The MigrationChunkStorageProvider is not intended for direct instantiation in application logic. It is designed to be deserialized from a world configuration file by the server's storage management system using its static CODEC field. This occurs once during the server's world-loading bootstrap sequence.

-   **Scope:** An instance of this provider is scoped to a single world session. It persists for the entire duration that a world is loaded on the server. It acts as a stateless configuration object after its initial creation.

-   **Destruction:** The object is eligible for garbage collection when the world is unloaded or the server shuts down. The `close` method on the loaders it creates is responsible for releasing any underlying file or database handles.

## Internal State & Concurrency
-   **State:** The internal state, consisting of the `from` array and the `to` provider, is **effectively immutable** after construction. The object is configured once during deserialization and its state is not modified thereafter.

-   **Thread Safety:** The MigrationChunkStorageProvider itself is thread-safe due to its immutable nature. However, the concurrency guarantees of the objects it creates are more nuanced:
    -   **MigrationChunkLoader:** The `loadHolder` method is fully asynchronous and non-blocking, built upon CompletableFuture. It is designed for high-throughput, concurrent invocation from multiple chunk-loading threads.
    -   **WARNING:** Methods such as `getIndexes` and `close` on the created loader are **not** thread-safe. They are expected to be called within a controlled, single-threaded context during specific phases of the world lifecycle, such as initialization or shutdown. Concurrent calls to these methods will result in undefined behavior and potential resource corruption.

## API Surface
The public contract is focused on its factory role as defined by the IChunkStorageProvider interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getLoader(store) | IChunkLoader | O(N) | Creates and returns a new MigrationChunkLoader instance. Complexity is linear based on the number of `from` providers. |
| getSaver(store) | IChunkSaver | O(1) | Creates and returns an IChunkSaver by delegating directly to the configured `to` provider. |

## Integration Patterns

### Standard Usage
The primary and intended method of integration is declarative, via world storage configuration. A server administrator defines the chain of loaders and the single saver in a configuration file that the server parses on startup.

```yaml
# Example world_storage.yml configuration
# The server's CODEC system would parse this and instantiate the provider.
storage:
  id: "Migration"
  loaders:
    - id: "RegionFile" # Attempt to load from legacy format first
      path: "world/region"
    - id: "LevelDB"   # Fallback to another format
      path: "world/leveldb"
  saver:
    id: "Anvil"       # All new and modified chunks are saved here
    path: "world/anvil"
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Avoid `new MigrationChunkStorageProvider()` in game logic. This bypasses the centralized configuration system and can lead to a fractured or incorrect storage setup. The provider's behavior is entirely dependent on its configuration, which should be managed externally.

-   **Stateful Providers:** Do not configure this provider with subordinate loaders or savers that rely on shared, mutable state between them. The migration provider assumes each underlying provider is independent.

-   **Read-After-Write Inconsistency:** Be aware that if a chunk is loaded from a legacy provider and immediately saved to the new provider, a subsequent load request within the same session might still read the old data until caches are invalidated or the server is restarted. The migration is eventual, not instantaneous across all systems.

## Data Pipeline
The provider splits the data pipeline into two distinct paths for reading and writing chunks.

**Read Path (Loading)**
> Flow:
> World Engine Chunk Request → `getLoader()` → MigrationChunkLoader → Attempt `from[0]`.load → (On Failure) → Attempt `from[1]`.load → ... → (On Success) → `CompletableFuture<Chunk>` → World Engine

**Write Path (Saving)**
> Flow:
> World Engine Chunk Data → `getSaver()` → `to`.getSaver() → IChunkSaver.save() → Final Storage Medium (e.g., Anvil File)

