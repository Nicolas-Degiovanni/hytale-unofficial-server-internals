---
description: Architectural reference for FutureUtils
---

# FutureUtils

**Package:** com.hypixel.hytale.builtin.hytalegenerator
**Type:** Utility

## Definition
```java
// Signature
public class FutureUtils {
```

## Architecture & Concepts
FutureUtils is a stateless, low-level concurrency primitive designed to simplify asynchronous task aggregation. It serves as a quality-of-life adapter between Java's standard collection framework (specifically List) and the `java.util.concurrent.CompletableFuture` API.

The core Java `CompletableFuture.allOf` method requires a varargs array of futures, which is often inconvenient when dealing with dynamically generated collections of asynchronous tasks. This utility class provides a clean, type-safe bridge, allowing engine components like the world generator or asset loader to easily synchronize a dynamic set of parallel operations. It is a foundational building block, not a high-level system, and is expected to be used by any component that orchestrates multiple concurrent computations.

### Lifecycle & Ownership
- **Creation:** Not applicable. FutureUtils is a static utility class and cannot be instantiated. Its methods are accessed statically.
- **Scope:** Static. The class and its methods are available for the entire lifetime of the ClassLoader that loaded it.
- **Destruction:** Not applicable. The class is unloaded along with its ClassLoader, typically upon application termination.

## Internal State & Concurrency
- **State:** Stateless. This class holds no internal state, either at the static or instance level. Its operations are pure functions, with outputs depending exclusively on their inputs.
- **Thread Safety:** Inherently thread-safe. As a stateless utility, its methods operate solely on their input arguments without side effects or reliance on shared mutable state. It can be safely invoked from any thread without external synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| allOf(List<CompletableFuture<T>> tasks) | CompletableFuture<Void> | O(n) | Aggregates a list of futures into a single future that completes only after every future in the input list has completed. Throws NullPointerException if the input list is null. |

## Integration Patterns

### Standard Usage
This utility is invoked when a system needs to wait for a dynamically sized group of asynchronous tasks to finish before proceeding. A common use case is parallel chunk generation, where the main thread must wait for all worker threads to complete their assigned chunks.

```java
// Example: Waiting for multiple asynchronous tasks to complete
List<CompletableFuture<ChunkData>> generationTasks = new ArrayList<>();
generationTasks.add(generator.generateChunkAsync(pos1));
generationTasks.add(generator.generateChunkAsync(pos2));
generationTasks.add(generator.generateChunkAsync(pos3));

// Use FutureUtils to create a single future that represents the entire batch
CompletableFuture<Void> allTasks = FutureUtils.allOf(generationTasks);

// Block until all generation tasks are finished
allTasks.join();

System.out.println("All chunks have been generated.");
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The class is not designed to be instantiated. Attempting to call `new FutureUtils()` will fail as no public constructor is provided. All access must be static.
- **Ignoring the Returned Future:** The `allOf` method is non-blocking. Simply calling the method does not wait for the tasks to complete. The returned `CompletableFuture<Void>` must be used to chain subsequent actions or to explicitly block execution.

```java
// BAD: This code does not wait for the tasks to finish
List<CompletableFuture<Void>> tasks = ...;
FutureUtils.allOf(tasks); // The returned future is discarded
// The program continues immediately, likely causing race conditions
```

## Data Pipeline
FutureUtils acts as a synchronization point in a control flow, not a data transformation pipeline. It aggregates multiple asynchronous control flows into a single, unified flow.

> Flow:
> Collection of active `CompletableFuture<T>` -> **FutureUtils.allOf()** -> Single aggregate `CompletableFuture<Void>` -> Downstream continuation logic (e.g., `thenRun()`, `join()`)

