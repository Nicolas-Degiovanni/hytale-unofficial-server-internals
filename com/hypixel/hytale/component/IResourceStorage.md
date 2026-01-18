---
description: Architectural reference for IResourceStorage
---

# IResourceStorage

**Package:** com.hypixel.hytale.component
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface IResourceStorage {
   @Nonnull
   <T extends Resource<ECS_TYPE>, ECS_TYPE> CompletableFuture<T> load(
      @Nonnull Store<ECS_TYPE> var1, @Nonnull ComponentRegistry.Data<ECS_TYPE> var2, @Nonnull ResourceType<ECS_TYPE, T> var3
   );

   @Nonnull
   <T extends Resource<ECS_TYPE>, ECS_TYPE> CompletableFuture<Void> save(
      @Nonnull Store<ECS_TYPE> var1, @Nonnull ComponentRegistry.Data<ECS_TYPE> var2, @Nonnull ResourceType<ECS_TYPE, T> var3, T var4
   );

   @Nonnull
   <T extends Resource<ECS_TYPE>, ECS_TYPE> CompletableFuture<Void> remove(
      @Nonnull Store<ECS_TYPE> var1, @Nonnull ComponentRegistry.Data<ECS_TYPE> var2, @Nonnull ResourceType<ECS_TYPE, T> var3
   );
}
```

## Architecture & Concepts
The IResourceStorage interface defines a formal contract for the asynchronous, persistent storage of game **Resources**. It serves as a critical abstraction layer, decoupling the core Entity-Component-System (ECS) from the underlying storage medium, which could be a local file system, a remote database, or a cloud-based object store.

The primary architectural driver for this interface is performance. All operations return a CompletableFuture, ensuring that I/O-bound tasks (reading from or writing to disk/network) do not block the main game loop. This prevents stuttering and maintains a responsive user experience, especially during operations like saving world data or loading player profiles.

By operating on generic Resource types, this interface provides a unified, type-safe mechanism for persisting any component data that is designated as a persistent resource within the game engine.

### Lifecycle & Ownership
As an interface, IResourceStorage itself has no lifecycle. The lifecycle described here pertains to its concrete implementations (e.g., FileSystemResourceStorage).

- **Creation:** A concrete implementation of IResourceStorage is instantiated once by the primary service container during engine bootstrap. It is a foundational service required for many higher-level systems.
- **Scope:** The service instance is a singleton that persists for the entire application session. It is shared across all game worlds and client contexts.
- **Destruction:** The instance is destroyed during the engine's graceful shutdown sequence. Implementations are expected to flush any pending write operations and release all file or network handles at this time.

**WARNING:** Abrupt termination of the application may result in the loss of data for any in-flight `save` operations that have not yet completed.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, any concrete implementation is inherently stateful, managing internal I/O thread pools, file handles, database connection pools, or write-through caches.
- **Thread Safety:** This contract is designed for concurrent access. Implementations **must be thread-safe**. Methods are expected to be callable from any thread, particularly the main game thread. The implementation is responsible for dispatching the actual I/O work to a background thread pool and safely publishing the result via the returned CompletableFuture.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load(...) | CompletableFuture<T> | I/O Bound | Asynchronously retrieves a Resource from persistent storage. The future completes exceptionally if the resource is not found or a read error occurs. |
| save(...) | CompletableFuture<Void> | I/O Bound | Asynchronously persists a Resource to storage, overwriting any existing data for that resource. The future completes exceptionally if a write error occurs. |
| remove(...) | CompletableFuture<Void> | I/O Bound | Asynchronously deletes a Resource from persistent storage. The future completes exceptionally if the resource does not exist or a deletion error occurs. |

## Integration Patterns

### Standard Usage
Interaction with this system should always be asynchronous. Chain callbacks to the returned CompletableFuture to process results or handle errors without blocking the calling thread.

```java
// Standard asynchronous load and process
IResourceStorage storage = context.getService(IResourceStorage.class);
CompletableFuture<PlayerData> future = storage.load(store, data, PLAYER_DATA_TYPE);

future.thenAccept(playerData -> {
    // This code executes on a worker thread when the data is ready
    System.out.println("Player data loaded for: " + playerData.getName());
}).exceptionally(error -> {
    // Handle cases where the load failed (e.g., file not found, corrupted)
    log.error("Failed to load player data", error);
    return null; // Return null to signify the exception was handled
});
```

### Anti-Patterns (Do NOT do this)
- **Blocking the Main Thread:** Never call `future.get()` or `future.join()` on a future returned by this service from a performance-critical thread like the main game loop. This completely negates the asynchronous design and will cause the game to freeze.
- **Ignoring Failures:** Do not initiate a `save` or `remove` operation without attaching an `exceptionally` handler. Silent I/O failures can lead to data corruption or loss, and this is the only mechanism to detect and react to them.
- **Direct Implementation:** Do not provide your own implementation of IResourceStorage in game logic code. Always retrieve the engine-provided singleton instance from the central service context.

## Data Pipeline
The IResourceStorage interface sits between the in-memory component system and the physical storage layer.

> **Save Flow:**
> Game Logic -> `IResourceStorage.save(Resource)` -> **Implementation** -> Serialization -> I/O Worker Thread -> Disk / Network Write

> **Load Flow:**
> Game Logic -> `IResourceStorage.load()` -> **Implementation** -> I/O Worker Thread -> Disk / Network Read -> Deserialization -> `CompletableFuture` Completion -> Game Logic Callback

