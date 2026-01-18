---
description: Architectural reference for CompletableFutureUtil
---

# CompletableFutureUtil

**Package:** com.hypixel.hytale.common.util
**Type:** Utility

## Definition
```java
// Signature
public class CompletableFutureUtil {
```

## Architecture & Concepts
CompletableFutureUtil is a static, stateless utility class designed to standardize and simplify asynchronous programming patterns across the Hytale engine. It directly addresses common boilerplate and potential pitfalls associated with Java's native CompletableFuture API.

This class acts as a centralized policy layer for asynchronous operations. Its primary responsibilities include:
- **Global Exception Handling:** Provides a default, engine-wide mechanism to catch, log, and propagate unhandled exceptions from asynchronous tasks, preventing silent failures that are notoriously difficult to debug.
- **Progress Aggregation:** Offers a blocking mechanism to monitor a collection of concurrent tasks and report their collective progress, a critical function for systems like asset loading or world generation that must update a user interface.
- **Cancellation Logic:** Simplifies the detection of task cancellation, which can be obscured by nested CompletionExceptions.

By providing these shared utilities, CompletableFutureUtil ensures that all asynchronous code in the engine behaves consistently, especially concerning error reporting and lifecycle management.

## Lifecycle & Ownership
As a static utility class, CompletableFutureUtil is not instantiated and therefore has no object lifecycle in the traditional sense.

- **Creation:** The class is loaded by the JVM ClassLoader at application startup or when first referenced. It is never instantiated.
- **Scope:** Its static methods are available globally throughout the application's lifetime.
- **Destruction:** The class is unloaded when the application's ClassLoader is garbage collected, which typically occurs at JVM shutdown.

## Internal State & Concurrency
CompletableFutureUtil is fundamentally stateless and thread-safe.

- **State:** This class holds no mutable state, either static or instance-based. All methods operate exclusively on the arguments provided to them. The static `fn` field is a constant, immutable lambda expression.
- **Thread Safety:** All methods are reentrant and safe to be called from any thread. The methods themselves do not introduce synchronization primitives. However, the caller is responsible for ensuring that the CompletableFuture and callback arguments are used in a thread-safe manner. The `joinWithProgress` method is explicitly designed to be called from a single thread that intends to block while other threads complete the provided futures.

## API Surface
The public API provides coarse-grained, high-level patterns for managing asynchronous workflows.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| _catch(future) | CompletableFuture | O(1) | Attaches the global exception handling logic to a future. This is the primary mechanism for ensuring uncaught exceptions are logged. |
| isCanceled(throwable) | boolean | O(N) | Recursively inspects an exception chain of depth N to determine if the root cause was a CancellationException. |
| joinWithProgress(list, callback, ...) | void | Blocking | Blocks the calling thread until all futures in the list are complete. Periodically invokes the progress callback on the calling thread. Throws InterruptedException. |
| whenComplete(future, callee) | CompletableFuture | O(1) | Chains two futures, ensuring the callee is completed (either successfully or exceptionally) when the first future completes. |
| completionCanceled() | CompletableFuture | O(1) | Returns a new, pre-canceled CompletableFuture. |

## Integration Patterns

### Standard Usage
The most common use case is to wrap an asynchronous operation with `_catch` to ensure proper logging and then use `joinWithProgress` on a dedicated loading thread to monitor a batch of such operations.

```java
// Example: Loading multiple assets on a background thread
List<CompletableFuture<Asset>> assetFutures = new ArrayList<>();
for (AssetKey key : keysToLoad) {
    CompletableFuture<Asset> future = assetService.loadAssetAsync(key);
    assetFutures.add(CompletableFutureUtil._catch(future)); // ALWAYS wrap with _catch
}

// On a dedicated loading thread, block and report progress to the UI
try {
    CompletableFutureUtil.joinWithProgress(
        assetFutures,
        (progress, done, total) -> {
            // This callback runs on the loading thread.
            // Dispatch UI updates to the main thread.
            uiService.updateLoadingBar(progress);
        },
        16,  // Poll interval
        100  // Progress update interval
    );
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    // Handle interruption
}
```

### Anti-Patterns (Do NOT do this)
- **Calling `joinWithProgress` on the Main Thread:** This method is **blocking**. Calling it on a critical thread like the main game loop or rendering thread will freeze the application. It must only be used on dedicated worker or loading threads.
- **Forgetting `_catch`:** Failing to wrap a CompletableFuture with `_catch` bypasses the engine's global exception logging. An exception thrown within that future may be silently swallowed, leading to deadlocks or undefined behavior.
- **Direct Instantiation:** This is a utility class. Do not attempt to create an instance with `new CompletableFutureUtil()`.

## Data Pipeline
CompletableFutureUtil primarily manages control flow rather than a data pipeline. Its two main flows are for exception handling and progress reporting.

**Exception Handling Flow:**
> Flow:
> Asynchronous Task -> CompletableFuture -> **CompletableFutureUtil._catch** -> HytaleLogger (on failure) -> TailedRuntimeException

**Progress Reporting Flow:**
> Flow:
> List of Futures -> **CompletableFutureUtil.joinWithProgress** (Blocking Poll) -> ProgressConsumer Callback -> UI System Update

