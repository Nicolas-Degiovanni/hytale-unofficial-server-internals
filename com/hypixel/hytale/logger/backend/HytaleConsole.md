---
description: Architectural reference for HytaleConsole
---

# HytaleConsole

**Package:** com.hypixel.hytale.logger.backend
**Type:** Singleton

## Definition
```java
// Signature
public class HytaleConsole extends Thread {
```

## Architecture & Concepts
The HytaleConsole class is a foundational component of the engine's logging subsystem, implemented as an active object. It operates as a dedicated, asynchronous worker thread responsible for processing and writing log records to the standard system output streams.

Its primary architectural purpose is to decouple log-producing threads (such as the main game loop or network threads) from the potentially blocking I/O operations of writing to a console. By employing a producer-consumer pattern with an internal `BlockingQueue`, HytaleConsole ensures that logging calls are non-blocking and have minimal performance impact on critical application threads. Log producers submit `LogRecord` objects, which are queued internally and processed sequentially by the HytaleConsole thread, guaranteeing ordered output and preventing log message interleaving.

This design is critical for maintaining application performance, especially in high-throughput logging scenarios, by offloading I/O latency to a low-priority background thread.

## Lifecycle & Ownership
- **Creation:** Instantiated eagerly as a static final singleton, `HytaleConsole.INSTANCE`, upon class-loading by the JVM. The internal worker thread is named "HytaleConsole", marked as a daemon, and started immediately within the private constructor.
- **Scope:** Application-scoped. The singleton instance persists for the entire lifetime of the JVM process. As a daemon thread, it will not prevent the application from exiting if all other non-daemon threads have completed.
- **Destruction:** Explicitly terminated via the `shutdown` method. This is a graceful shutdown sequence that interrupts the worker thread, waits for its completion, and synchronously processes any remaining queued log records before flushing and closing the underlying output streams.

**WARNING:** Failure to call `shutdown` during a controlled application exit may result in the loss of buffered log messages, as the daemon thread will be terminated abruptly by the JVM.

## Internal State & Concurrency
- **State:** Mutable. The core state is the `LinkedBlockingQueue` of `LogRecord` objects, which acts as the central work buffer. The class also maintains state for the underlying `OutputStreamWriter` instances for `System.out` and `System.err`, and a `terminalType` string which determines whether ANSI color codes should be rendered.
- **Thread Safety:** Thread-safe. This class is designed to be safely accessed from multiple threads. The public `publish` method uses the concurrent `BlockingQueue.offer` operation to safely enqueue log records from any producer thread. All internal state modification and I/O operations, such as formatting and writing to the console, are confined to the single, dedicated worker thread, eliminating the need for explicit locks on internal resources.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| publish(LogRecord) | void | O(1) | Submits a log record for asynchronous processing. This is a non-blocking operation. |
| shutdown() | void | O(N) | Initiates a graceful shutdown. Blocks the calling thread until the worker is terminated and all N remaining logs are flushed. |
| setTerminal(String) | void | O(1) | Configures the terminal type to enable or disable ANSI color support. |
| getFormatter() | HytaleLogFormatter | O(1) | Returns the formatter instance used by this console backend. |

## Integration Patterns

### Standard Usage
HytaleConsole is a low-level backend component and is not intended for direct use by application code. It is managed by the higher-level logging facade. The typical interaction is indirect, where the logging system retrieves the singleton instance and uses it as a log handler.

```java
// Internal engine code obtaining the singleton instance
HytaleConsole console = HytaleConsole.INSTANCE;

// A log record created elsewhere in the system
LogRecord record = new LogRecord(Level.INFO, "Server has started successfully.");

// Publishing the record to the queue for asynchronous writing
console.publish(record);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The class is a singleton with a private constructor. Attempting to instantiate it via reflection will break the logging subsystem and lead to unpredictable behavior. Always use the static `INSTANCE` field.
- **Assuming Synchronous Writes:** Do not write code that assumes a log message has been physically written to the console immediately after `publish` returns. The call only enqueues the message; the actual write happens at a later time on a separate thread.
- **Modifying Formatter State:** Retrieving the `HytaleLogFormatter` via `getFormatter` and modifying its state while the console is running is not thread-safe and can lead to corruption of log output.

## Data Pipeline
The flow of a single log message through this component is linear and asynchronous, from initial submission to final output.

> Flow:
> LogRecord Object -> `HytaleConsole.publish()` -> `BlockingQueue` -> **HytaleConsole Worker Thread** -> `HytaleLogFormatter` -> `System.out` / `System.err`

