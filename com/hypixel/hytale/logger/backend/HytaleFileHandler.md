---
description: Architectural reference for HytaleFileHandler
---

# HytaleFileHandler

**Package:** com.hypixel.hytale.logger.backend
**Type:** Singleton

## Definition
```java
// Signature
public class HytaleFileHandler extends Thread {
```

## Architecture & Concepts
The HytaleFileHandler is a specialized, thread-based backend for the Hytale logging system. Its primary architectural purpose is to decouple log-producing application threads from the high-latency, blocking I/O operations of writing log messages to a disk file.

This class implements the **Producer-Consumer** pattern. Application threads act as producers, submitting LogRecord objects via the log method. The HytaleFileHandler itself is the single consumer, running on its own dedicated thread ("HytaleLogger"). It continuously pulls records from a concurrent queue and delegates the actual file writing to a standard Java FileHandler.

This design ensures that application threads (such as the main game loop or network threads) do not stall while waiting for disk I/O, significantly improving performance and responsiveness. The class is implemented as an eager-initialized Singleton, providing a single, globally accessible point for file logging management.

### Lifecycle & Ownership
The lifecycle of this handler is critical and must be managed explicitly by the application's entry point.

-   **Creation:** The singleton INSTANCE is instantiated by the JVM during class loading. However, it remains in a dormant state.
-   **Scope:** The singleton instance persists for the entire lifetime of the application. The internal logging thread and file resources, however, are only active between calls to enable and shutdown.
-   **Destruction:** The shutdown method must be called during the application's shutdown sequence. This method gracefully terminates the logging thread, drains any remaining messages from the queue, and flushes all buffered output to the disk. **WARNING:** Failure to call shutdown will result in the loss of any pending log messages.

## Internal State & Concurrency
-   **State:** The HytaleFileHandler maintains mutable state, including the nullable FileHandler reference and the BlockingQueue of LogRecord objects. Its state transitions through distinct phases: uninitialized, enabled (running), and shutdown.

-   **Thread Safety:** This class is designed for high-concurrency environments.
    -   The public log method is **thread-safe**. It uses a LinkedBlockingQueue, a concurrent data structure, to safely accept LogRecord objects from any number of producer threads without explicit locking.
    -   The lifecycle methods, enable and shutdown, are **not thread-safe** and are not re-entrant. They are designed to be called once from a single, controlling thread at application startup and shutdown, respectively.

## API Surface
The public API is focused on lifecycle management. The primary data-ingestion method, log, is intended to be called by higher-level logging facades.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| enable() | void | O(1) + I/O | Initializes and starts the logging thread. Creates the log directory and file. Throws IllegalStateException if already enabled. |
| log(LogRecord) | void | O(1) | Enqueues a log record for asynchronous processing. This is the primary, high-frequency entry point. |
| shutdown() | void | O(N) | Gracefully stops the logging thread, drains the queue of N remaining items, and closes the file. |
| getFileHandler() | FileHandler | O(1) | Returns the underlying Java FileHandler, or null if not enabled. |

## Integration Patterns

### Standard Usage
The HytaleFileHandler must be explicitly enabled at application startup and shut down before termination. It is not designed to be used directly for logging; a logging facade should be used to create and submit LogRecord objects.

```java
// Application Startup
// Retrieve the global singleton instance
HytaleFileHandler fileLogger = HytaleFileHandler.INSTANCE;

try {
    // Activate the logging thread and open the log file
    fileLogger.enable();
} catch (RuntimeException e) {
    // Handle failure to initialize logging
    System.err.println("Fatal: Could not start file logger.");
    // Terminate application
}

// ... Application runs, logging occurs via a facade which calls fileLogger.log()

// Application Shutdown
fileLogger.shutdown();
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new HytaleFileHandler()`. This violates the singleton pattern and will result in an unmanaged, non-functional instance. Always use `HytaleFileHandler.INSTANCE`.
-   **Logging Before Enabling:** Calling the log method before enable has been successfully completed is not a supported use case. While the current implementation has a fallback to publish directly, this bypasses the asynchronous queue and blocks the calling thread, defeating the purpose of the class.
-   **Multiple Enable Calls:** Calling enable more than once on the instance will throw an IllegalStateException. The handler is not designed to be restarted.
-   **Forgetting Shutdown:** Failure to call shutdown will result in an abrupt termination of the logger thread. Any log messages remaining in the queue will be lost permanently.

## Data Pipeline
The HytaleFileHandler serves as a specific stage in the engine's logging data pipeline, acting as an asynchronous buffer to the filesystem.

> Flow:
> Log Facade -> LogRecord Creation -> **HytaleFileHandler.log()** -> Enqueue to BlockingQueue -> HytaleLogger Thread Dequeues -> `java.util.logging.FileHandler.publish()` -> Formatter -> Disk I/O (Log File)

