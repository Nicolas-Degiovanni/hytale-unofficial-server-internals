---
description: Architectural reference for Handle
---

# Handle

**Package:** com.hypixel.hytale.builtin.hytalegenerator.plugin
**Type:** Stateful Service

## Definition
```java
// Signature
public class Handle implements IWorldGen {
```

## Architecture & Concepts
The Handle class is a critical component that acts as a bridge between the server's abstract world generation engine and the concrete implementation provided by the HytaleGenerator plugin. It implements the standard IWorldGen interface, allowing it to be seamlessly registered with and controlled by the core server.

Its primary role is to translate generic world generation requests from the server into specific, asynchronous tasks understood by the HytaleGenerator plugin. Each Handle instance is configured with a specific GeneratorProfile, effectively representing a single, configured world generator. This design allows multiple, distinct world generation algorithms and configurations to coexist and be managed by the server through a common interface.

This class embodies the **Strategy Pattern**, where the server's world engine is the context, IWorldGen is the strategy interface, and Handle is a concrete strategy implementation.

### Lifecycle & Ownership
- **Creation:** A Handle is not created directly by end-users. It is instantiated by its parent HytaleGenerator plugin, which bundles it with a specific GeneratorProfile. The plugin then registers this Handle instance with the server's world management system.
- **Scope:** The lifecycle of a Handle is tightly coupled to the world it is configured to generate. It persists as long as the server's world engine maintains a reference to it for a given world or dimension.
- **Destruction:** The object is eligible for garbage collection when the associated world is unloaded and the server's world generation registry releases its reference.

## Internal State & Concurrency
- **State:** The Handle is a stateful object. It maintains a reference to a mutable GeneratorProfile. The `generate` method directly modifies this profile by calling `setSeed`, tailoring it for each specific request. This makes a Handle instance specific to a single, ongoing generation process.

- **Thread Safety:** This class is **not thread-safe**. The `generate` method's mutation of the shared GeneratorProfile introduces a potential race condition. If multiple threads were to invoke `generate` on the same Handle instance concurrently, the seed state would become corrupted. The design implicitly relies on the calling world engine to serialize requests to a single Handle instance or to create a new Handle for each distinct world generation context. The asynchronous nature of the `generate` method, which returns a CompletableFuture, ensures that the heavy lifting of chunk generation is performed off the main thread, but the initial request submission is not safe for concurrent access.

## API Surface
The public API is defined by the IWorldGen interface, focusing exclusively on world generation tasks.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generate(seed, index, x, z, stillNeeded) | CompletableFuture | O(1) | Submits an asynchronous chunk generation request to the HytaleGenerator plugin. The method itself is a lightweight dispatcher; the actual generation is a complex, long-running task. |
| getSpawnPoints(seed) | Transform[] | O(1) | Retrieves the predefined spawn location from the associated GeneratorProfile. |

## Integration Patterns

### Standard Usage
The Handle is not intended for direct use. The server's world engine interacts with it through the IWorldGen interface after it has been registered by the HytaleGenerator plugin.

```java
// Hypothetical server-side usage
IWorldGen worldGenerator = worldRegistry.getGeneratorFor("overworld");

// The server requests a chunk, which is dispatched to the Handle instance
CompletableFuture<GeneratedChunk> futureChunk = worldGenerator.generate(
    world.getSeed(),
    chunkIndex,
    chunkX,
    chunkZ,
    stillNeededPredicate
);

futureChunk.thenAccept(chunk -> {
    // Process the completed chunk
});
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create a Handle using `new Handle()`. The HytaleGenerator plugin is responsible for its creation and lifecycle management. Manually creating one would leave it disconnected from the required plugin infrastructure.
- **State Reuse:** Do not attempt to reuse a single Handle instance across different worlds or concurrent generation tasks. Its internal state (the GeneratorProfile) is not designed for this and will lead to unpredictable behavior.
- **Concurrent Invocation:** Do not call `generate` from multiple threads on the same Handle instance. This will cause a race condition on the internal seed state.

## Data Pipeline
The Handle serves as the entry point for a data pipeline that transforms a server request into a generated game chunk.

> Flow:
> Server World Engine Call -> **Handle.generate()** -> ChunkRequest Object Creation -> HytaleGenerator.submitChunkRequest() -> Asynchronous Worker Pool -> GeneratedChunk -> CompletableFuture Resolution -> Server World Engine

