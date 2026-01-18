---
description: Architectural reference for WorldPregenerateSystem
---

# WorldPregenerateSystem

**Package:** com.hypixel.hytale.server.core.universe.world.system
**Type:** Transient System Component

## Definition
```java
// Signature
public class WorldPregenerateSystem extends StoreSystem<ChunkStore> {
```

## Architecture & Concepts
The WorldPregenerateSystem is a specialized, one-shot system responsible for ensuring a predefined area of a world is loaded and generated before the world becomes fully active. It acts as a bootstrap task that runs during the server's world initialization sequence.

Its primary architectural function is to bridge server configuration with the asynchronous chunk loading mechanism. By reading a `pregenerateRegion` from the `WorldConfig`, it translates a high-level requirement ("ensure this area exists") into a batch of low-level, parallelizable tasks submitted to the `ChunkStore`.

This system is a critical component for server operators who need to guarantee that players spawning into a new world do not experience lag or delays associated with initial terrain generation. It operates based on a "fire-and-forget" model; once added to its parent `Store`, it initiates the pre-generation process and logs the result upon completion without requiring further intervention. Its dependency on the `ChunkStore.INIT_GROUP` ensures it executes only after the core chunk management infrastructure is fully operational.

### Lifecycle & Ownership
- **Creation:** The system is instantiated and added to the `ChunkStore`'s system registry by a higher-level authority, typically the `World` factory or loader, during the server's world setup phase. Its logic is not triggered by its constructor, but by the framework's `onSystemAddedToStore` callback.
- **Scope:** The instance persists as long as the `ChunkStore` it is registered with is active. However, its primary workload is finite and executes only once when it is first added to the store. It does not perform any per-tick updates.
- **Destruction:** The system is eligible for garbage collection when the parent `World` and its associated `ChunkStore` are unloaded. The `onSystemRemovedFromStore` method is empty, signifying that no explicit resource cleanup is necessary.

## Internal State & Concurrency
- **State:** The WorldPregenerateSystem is **stateless**. It maintains no instance-level fields that hold data between invocations. All operational variables, such as the list of futures and the completion counter, are confined to the local method scope of `onSystemAddedToStore`.
- **Thread Safety:** This class is inherently thread-safe. The core pre-generation logic is highly concurrent, dispatching numerous asynchronous chunk-loading tasks. It safely manages the aggregation of these parallel operations by collecting `CompletableFuture` objects and using an `AtomicInteger` for the completion counter. This is a standard, non-blocking pattern for tracking the progress of a batch of asynchronous tasks.

## API Surface
The public API is exclusively for integration with the `StoreSystem` framework and is not intended for direct developer consumption.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDependencies() | Set | O(1) | Returns the dependency set, ensuring this system runs after the `ChunkStore` is initialized. |
| onSystemAddedToStore(store) | void | O(N) | **Entry Point.** Framework callback that triggers the entire pre-generation process. N is the number of chunk regions to generate. |
| onSystemRemovedFromStore(store) | void | O(1) | Framework callback for cleanup. This implementation is a no-op. |

## Integration Patterns

### Standard Usage
This system is not designed to be used directly. It is enabled implicitly by defining a `pregenerateRegion` within the world's configuration file. The server's world loading process will then automatically add this system to the `ChunkStore`.

```java
// Conceptual example of how the framework adds the system
// This code does NOT belong in typical game logic.
ChunkStore chunkStore = world.getChunkStore();
chunkStore.addSystem(new WorldPregenerateSystem()); // This triggers the pre-generation
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WorldPregenerateSystem()` and attempt to manage it yourself. Its entire lifecycle and dependency resolution are managed by the `Store` it is added to.
- **Manual Invocation:** Never call `onSystemAddedToStore` directly. Doing so bypasses the framework's dependency management and will lead to race conditions or `NullPointerException` if the `ChunkStore` is not in a valid state.

## Data Pipeline
The system orchestrates a one-way data and command flow, transforming a configuration setting into a large-scale, asynchronous world generation task.

> Flow:
> `WorldConfig` -> `WorldPregenerateSystem.onSystemAddedToStore` -> Dispatches N calls to `ChunkStore.getChunkReferenceAsync` -> Parallel Chunk I/O & Generation on Worker Threads -> `CompletableFuture` Callbacks -> `HytaleLogger`

