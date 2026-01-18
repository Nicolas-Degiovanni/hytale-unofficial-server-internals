---
description: Architectural reference for HytaleLoggerBackend
---

# HytaleLoggerBackend

**Package:** com.hypixel.hytale.logger.backend
**Type:** Managed Singleton Registry

## Definition
```java
// Signature
public class HytaleLoggerBackend extends LoggerBackend {
```

## Architecture & Concepts
The HytaleLoggerBackend is the central nervous system for all logging operations within the Hytale engine. It serves as a custom implementation for Google's Flogger logging API, intercepting every log message generated throughout the application and routing it to the appropriate destinations.

Architecturally, it implements a hierarchical, tree-based logger system. Each logger instance is identified by a unique name (e.g., *com.hypixel.hytale.client.rendering*) and maintains a reference to a parent logger. This structure culminates in a single, static **ROOT_LOGGER**, which acts as the final destination for all log records.

Log messages propagate upwards from the originating child logger to its parent, and so on, until they reach the root. The ROOT_LOGGER is responsible for dispatching the final LogRecord to concrete handlers:
*   **HytaleFileHandler:** Persists logs to disk.
*   **HytaleConsole:** Prints formatted logs to the standard output stream.
*   **Sentry Integration:** Forwards exceptions and high-severity logs to the Sentry error reporting service.
*   **Dynamic Subscribers:** A list of in-memory listeners that can be attached at runtime for real-time log processing, often used by developer tools or in-game consoles.

This design decouples the log-producing code (which only needs to know about the Flogger API) from the log-consuming and routing infrastructure, which is entirely managed by this class.

### Lifecycle & Ownership
- **Creation:** The ROOT_LOGGER is a static final instance, created when the HytaleLoggerBackend class is loaded by the JVM. All other logger instances are created on-demand via the static `getLogger(String name)` factory method. When a logger for a new name is requested, it is instantiated and placed into a static, concurrent cache.
- **Scope:** All logger instances are application-scoped. Once created, they persist in the static cache for the entire lifetime of the application. This ensures that logger configuration (like log levels) is maintained.
- **Destruction:** There is no explicit destruction or cleanup mechanism. Loggers are held as strong references in the static cache and are only eligible for garbage collection when the application's class loader is unloaded, which effectively occurs at JVM shutdown.

## Internal State & Concurrency
- **State:** This class is highly stateful. Each instance maintains its name, parent, and configured log Level. The class itself manages significant shared static state, including the CACHE of all loggers, the ROOT_LOGGER, and the list of subscribers.
- **Thread Safety:** The backend is designed to be fully thread-safe and handle high-throughput logging from multiple application threads.
    - The central logger CACHE uses a **ConcurrentHashMap** to prevent race conditions during on-demand logger creation.
    - The list of dynamic subscribers on the ROOT_LOGGER uses a **CopyOnWriteArrayList**, which provides thread-safe iteration and modification, optimized for scenarios where reads (logging) are far more frequent than writes (subscribing/unsubscribing).

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getLogger(String name) | HytaleLoggerBackend | O(1) | Factory method to retrieve or create a logger instance for the given name. This is the primary entry point. |
| log(LogRecord record) | void | O(D) | Processes a single log record, forwarding it to its parent. D is the depth of the logger in the hierarchy. |
| isLoggable(Level lvl) | boolean | O(1) | A performance-critical check to determine if a message at the given level should be processed. |
| setLevel(Level newLevel) | void | O(1) | Atomically sets the minimum log level for this logger instance. |
| setSentryClient(IScopes scope) | void | O(1) | Attaches a Sentry client to this logger for error reporting. Can be null to disable. |
| subscribe(list) | void | O(N) | **Static.** Registers a list to receive all log records that reach the root. N is the number of existing subscribers. |
| reloadLogLevels() | void | O(M) | **Static.** Iterates all cached loggers and re-applies their log level from the configured LOG_LEVEL_LOADER. M is the number of cached loggers. |

## Integration Patterns

### Standard Usage
Developers should never interact with this class directly for producing logs. Instead, they should use the standard Flogger API. The backend is configured once at application startup. For administrative tasks, such as dynamically changing a log level, use the static methods.

```java
// Correctly retrieve a logger instance for administrative purposes.
// This does NOT produce a log message; it configures the logger.
HytaleLoggerBackend rendererLogger = HytaleLoggerBackend.getLogger("com.hypixel.hytale.client.rendering");

// Change the log level at runtime, for example, in response to a debug command.
rendererLogger.setLevel(java.util.logging.Level.FINEST);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructors are protected and should not be called directly. `new HytaleLoggerBackend("name")` will create a disconnected logger that does not participate in the global hierarchy or caching. Always use the static `getLogger` methods.
- **Manual Log Propagation:** Do not call the `log` method directly. This bypasses the Flogger API's rich formatting and metadata capabilities and can lead to malformed log records.
- **Blocking Subscribers:** Any code in a custom subscriber list should be extremely fast and non-blocking. Since subscribers are invoked synchronously on the logging thread, a slow subscriber will degrade the performance of the entire application.

## Data Pipeline
The flow of a log message through the system is a multi-stage process, starting from the application code and ending at a physical destination like a file or console.

> Flow:
> Flogger API Call -> Google Flogger Frontend -> **HytaleLoggerBackend (Child)** -> Sentry Handler (optional) -> **HytaleLoggerBackend (Parent)** -> ... -> **HytaleLoggerBackend (ROOT)**
>
> **ROOT_LOGGER** then dispatches concurrently to:
> 1. -> HytaleFileHandler -> Log File on Disk
> 2. -> HytaleConsole -> Standard Out/Err
> 3. -> All registered Subscribers -> In-Memory Lists

