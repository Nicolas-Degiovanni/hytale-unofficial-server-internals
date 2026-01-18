---
description: Architectural reference for ChunkRequest
---

# ChunkRequest

**Package:** com.hypixel.hytale.builtin.hytalegenerator.chunkgenerator
**Type:** Data Structure / DTO

## Definition
```java
// Signature
public record ChunkRequest(@Nonnull ChunkRequest.GeneratorProfile generatorProfile, @Nonnull ChunkRequest.Arguments arguments) {
```

## Architecture & Concepts

The ChunkRequest is a foundational data structure in the world generation pipeline. It is not an active service but a passive, immutable **command object**. Its primary role is to encapsulate all parameters required by a generator worker to produce a single world chunk. This design decouples the entity requesting a chunk (e.g., a world streaming system) from the asynchronous worker that fulfills the request.

The structure is composed of two distinct parts:
*   **GeneratorProfile:** Defines the high-level properties of the *world* being generated, such as its structural ruleset (worldStructureName), spawn point, and seed. This profile is shared across many chunk requests for the same world.
*   **Arguments:** Defines the specific parameters for the *individual chunk* being requested, including its coordinates (x, z) and a unique request identifier (index).

A critical architectural feature is the `stillNeeded` predicate within the Arguments record. This is a callback mechanism that allows the generator worker to periodically check if the requested chunk is still required by the game. This enables an efficient cancellation strategy, preventing the system from wasting CPU cycles generating chunks for areas the player has already left.

## Lifecycle & Ownership

-   **Creation:** A ChunkRequest is instantiated on-demand by a high-level world management system, such as a `ChunkProvider` or `WorldStreamer`, whenever a new chunk enters a player's view distance or is otherwise required.
-   **Scope:** The object is short-lived and transaction-scoped. It exists only for the duration of a single generation task. It is created, passed to a generation queue, consumed by a worker, and then becomes eligible for garbage collection.
-   **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or explicit cleanup methods associated with this object.

## Internal State & Concurrency

-   **State:** The top-level ChunkRequest record and its nested Arguments record are immutable by definition. However, the nested GeneratorProfile class is **mutable** due to the presence of the `setSeed` method.

    **WARNING:** The mutability of GeneratorProfile is a significant design consideration. Modifying a GeneratorProfile instance after it has been used to create a ChunkRequest can lead to severe, difficult-to-diagnose generation bugs and data corruption.

-   **Thread Safety:** This object is **not thread-safe** if the GeneratorProfile is modified. It is designed to be constructed on a main thread or management thread and then passed to a worker thread for read-only consumption. It must be treated as *effectively immutable* after its creation and submission to a generation queue. Any modification after submission constitutes a race condition.

## API Surface

The primary API consists of the record's constructor and its component accessors.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generatorProfile() | GeneratorProfile | O(1) | Returns the world-level generation profile. |
| arguments() | Arguments | O(1) | Returns the arguments for this specific chunk request. |
| arguments.stillNeeded() | LongPredicate | O(1) | Returns the cancellation predicate. May be null. |
| generatorProfile.setSeed(int) | void | O(1) | **DANGEROUS:** Mutates the seed within the profile. |

## Integration Patterns

### Standard Usage

A ChunkRequest should be constructed with all necessary data and immediately submitted to a generation service or worker queue. The `stillNeeded` predicate is essential for cooperative cancellation in an asynchronous environment.

```java
// Standard pattern for requesting a chunk
// 1. Obtain the world's generator profile
GeneratorProfile profile = world.getGeneratorProfile();

// 2. Define arguments for the specific chunk
long requestIndex = requestCounter.incrementAndGet();
ChunkRequest.Arguments args = new ChunkRequest.Arguments(
    profile.seed(),
    requestIndex,
    chunkX,
    chunkZ,
    (idx) -> world.isRequestStillValid(idx) // Cancellation check
);

// 3. Create the command object and submit it
ChunkRequest request = new ChunkRequest(profile, args);
chunkGeneratorService.submit(request);
```

### Anti-Patterns (Do NOT do this)

-   **State Mutation After Submission:** Never modify the GeneratorProfile after the ChunkRequest has been created and passed to another system. This breaks the command pattern and introduces severe race conditions.

    ```java
    // ANTI-PATTERN: Modifying state after submission
    ChunkRequest request = new ChunkRequest(profile, args);
    chunkGeneratorService.submit(request);
    
    // This will cause undefined behavior in the generator worker
    profile.setSeed(12345); 
    ```

-   **Reusing Arguments:** Do not reuse an Arguments object for multiple requests. Each request must have a unique index and context.

-   **Ignoring Cancellation:** Passing a null or a constant `true` predicate to the `stillNeeded` argument bypasses the engine's cancellation mechanism, potentially leading to significant performance degradation under load as the system will perform work on unneeded chunks.

## Data Pipeline

The ChunkRequest serves as the data payload that flows from the world management layer to the core generation workers.

> Flow:
> World Streaming Manager -> `new ChunkRequest(...)` -> Chunk Generation Queue -> **ChunkRequest** (consumed by worker) -> Generator Worker -> Generated Chunk Data -> World State

