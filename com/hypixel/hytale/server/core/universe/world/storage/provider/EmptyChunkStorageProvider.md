---
description: Architectural reference for EmptyChunkStorageProvider
---

# EmptyChunkStorageProvider

**Package:** com.hypixel.hytale.server.core.universe.world.storage.provider
**Type:** Singleton

## Definition
```java
// Signature
public class EmptyChunkStorageProvider implements IChunkStorageProvider {
```

## Architecture & Concepts
The EmptyChunkStorageProvider is a specific implementation of the IChunkStorageProvider interface that serves as a **Null Object**. Its primary architectural function is to provide a non-operational, or "no-op", storage backend for the world engine.

This component is used in scenarios where chunk persistence is explicitly disabled. This includes temporary worlds, certain test environments, or game modes where the world state is entirely transient and should not be written to disk.

By providing a concrete implementation that does nothing, the system avoids conditional logic and null checks throughout the world management code. The engine can always interact with a valid IChunkStorageProvider, which simply discards save requests and fails all load requests. This makes it a "black hole" for chunk data, ensuring no I/O operations occur.

## Lifecycle & Ownership
- **Creation:** The EmptyChunkStorageProvider is a static singleton. A single instance, named INSTANCE, is created at class-loading time. It is not managed by a dependency injection framework or created during the server bootstrap sequence.
- **Scope:** Application-scoped. The single INSTANCE persists for the entire lifetime of the server process.
- **Destruction:** The instance is not explicitly destroyed. It is eligible for garbage collection only when the application's class loader is unloaded, which typically occurs at JVM shutdown.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no fields that store world data, caching information, or any other mutable state. Its behavior is constant and predictable.
- **Thread Safety:** The EmptyChunkStorageProvider is inherently **thread-safe**. As a stateless object, it can be safely shared and accessed by multiple threads concurrently without any need for locks or synchronization primitives. All methods return pre-completed futures or static instances, eliminating any risk of race conditions within this component.

## API Surface
The public contract is minimal, focused on providing the no-op loader and saver components.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getLoader(Store store) | IChunkLoader | O(1) | Returns a static, shared instance of a loader that always fails to find chunks. |
| getSaver(Store store) | IChunkSaver | O(1) | Returns a static, shared instance of a saver that silently discards all chunk data. |

## Integration Patterns

### Standard Usage
This provider is typically selected via server or world configuration, where the storage provider ID is set to "Empty". The world engine then retrieves the singleton instance and uses it to manage chunk I/O, effectively disabling persistence for that world.

```java
// Example of how the engine might be configured (conceptual)
WorldConfiguration config = new WorldConfiguration();
config.setStorageProvider(EmptyChunkStorageProvider.INSTANCE);

// The world engine proceeds without needing to know persistence is disabled
World world = new World(config);
world.saveChunk(chunk); // This operation will do nothing
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new EmptyChunkStorageProvider()`. This violates the singleton pattern and creates unnecessary objects. Always use the static `EmptyChunkStorageProvider.INSTANCE` field.
- **Expecting Persistence:** **WARNING:** Configuring this provider for a world that requires persistence will result in silent and irreversible data loss. Any progress made in the world will be lost upon shutdown. This provider should only be used when transient world behavior is the explicit goal.

## Data Pipeline
The EmptyChunkStorageProvider acts as a terminal node in the data pipeline, effectively halting any data flow to a persistent medium.

**Save Operation Flow:**
> Flow:
> World Engine -> Chunk Data -> IChunkSaver -> **EmptyChunkStorageProvider** -> Discarded

**Load Operation Flow:**
> Flow:
> World Engine -> Chunk Request -> IChunkLoader -> **EmptyChunkStorageProvider** -> Fails (returns null) -> World Generator (creates new chunk)

