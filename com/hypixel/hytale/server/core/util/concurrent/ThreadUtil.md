---
description: Architectural reference for ThreadUtil
---

# ThreadUtil

**Package:** com.hypixel.hytale.server.core.util.concurrent
**Type:** Utility

## Definition
```java
// Signature
public class ThreadUtil {
```

## Architecture & Concepts
ThreadUtil is a static, low-level utility class that provides foundational, server-wide concurrency primitives. It does not participate directly in the game loop or business logic; instead, it acts as a centralized factory and manager for creating and configuring threads and executor services according to engine-specific patterns.

Its primary role is to abstract and standardize common but complex threading operations. This includes:
- **Standardized Thread Pools:** Providing pre-configured ExecutorService instances for common server workloads, ensuring consistency in thread lifecycle and resource management.
- **Daemon Thread Factories:** Simplifying the creation of daemon threads, which do not prevent the JVM from exiting, for background tasks.
- **System-Level Timer Manipulation:** Influencing the underlying operating system scheduler to achieve higher-precision timing, which is critical for server tick-rate stability.
- **JVM Lifecycle Control:** Offering explicit mechanisms to keep the application alive, a pattern often required in server environments that might otherwise exit prematurely if only daemon threads are running.

The internal ThreadWatcher class is a highly specialized diagnostic tool that leverages the Java SecurityManager to enforce architectural rules about thread creation at runtime.

## Lifecycle & Ownership
- **Creation:** As a class containing only static methods, ThreadUtil is never instantiated. Its bytecode is loaded by the JVM ClassLoader when one of its methods is first invoked.
- **Scope:** The utility methods are available for the entire lifetime of the JVM.
- **Destruction:** The class is unloaded when the JVM shuts down. There is no instance-level cleanup.

## Internal State & Concurrency
- **State:** ThreadUtil is entirely stateless. Each method call operates independently and does not modify any internal fields of the class itself. Methods that return objects, such as newCachedThreadPool or daemonCounted, create *new* stateful objects (ThreadPoolExecutor, ThreadFactory with an AtomicLong) for the caller to manage.
- **Thread Safety:** All methods in this class are inherently thread-safe. They can be called from any thread at any time without external synchronization. The objects returned by the factory methods, such as ExecutorService, adhere to their own documented concurrency guarantees.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| forceTimeHighResolution() | void | O(1) | Spawns a long-lived daemon thread to request a high-resolution system timer from the OS. This is a system-wide side effect. |
| createKeepAliveThread(Semaphore) | void | O(1) | Creates a non-daemon thread that blocks indefinitely on the provided Semaphore. This prevents the JVM from exiting. |
| newCachedThreadPool(int, ThreadFactory) | ExecutorService | O(1) | Creates a cached thread pool optimized for bursty, short-lived tasks. |
| daemon(String) | ThreadFactory | O(1) | Returns a factory that creates daemon threads with a fixed name. |
| daemonCounted(String) | ThreadFactory | O(1) | Returns a factory that creates daemon threads with a formatted, auto-incrementing name. |

## Integration Patterns

### Standard Usage
ThreadUtil is the canonical source for creating managed thread pools within the server environment. Services should use it to provision background workers.

```java
// Create a thread pool for background asset processing
ThreadFactory factory = ThreadUtil.daemonCounted("AssetProcessor-%d");
ExecutorService assetExecutor = ThreadUtil.newCachedThreadPool(16, factory);

// Submit a task
assetExecutor.submit(() -> {
    // ... process assets
});
```

### Anti-Patterns (Do NOT do this)
- **Redundant Timer Resolution:** Do not call forceTimeHighResolution more than once. The first call spawns a thread that persists for the application's lifetime. Subsequent calls are wasteful and create unnecessary threads.
- **Leaking Keep-Alive Threads:** The Semaphore passed to createKeepAliveThread is the only mechanism to terminate the created thread. Failure to release the semaphore permit during server shutdown will result in a hung process that cannot exit gracefully.
- **Using ThreadWatcher in Production:** The internal ThreadWatcher class relies on the Java SecurityManager, which is deprecated and incurs significant performance overhead. It is a powerful diagnostic tool intended *only* for debugging and should never be enabled in a production environment.

## Data Pipeline
ThreadUtil is not a component in a data pipeline; it is a foundational utility that *builds* the components that execute pipeline stages. It provides the execution context (the threads and pools) where data processing occurs.

> **Example Flow Enabled by ThreadUtil:**
>
> Network Packet -> Netty I/O Thread -> Task Queue -> **ThreadPool (from ThreadUtil)** -> World Simulation Update

