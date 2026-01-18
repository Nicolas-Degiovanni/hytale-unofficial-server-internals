---
description: Architectural reference for HytaleLogger
---

# HytaleLogger

**Package:** com.hypixel.hytale.logger
**Type:** Utility / Factory

## Definition
```java
// Signature
public class HytaleLogger extends AbstractLogger<HytaleLogger.Api> {
```

## Architecture & Concepts
The HytaleLogger is the centralized logging facade for the entire Hytale application. It is not merely a logging utility but a foundational service that aggressively integrates with the Java Virtual Machine to capture all log and console output. It is built upon Google's Flogger library, inheriting its high-performance, fluent API design.

Its primary architectural function is to act as a single, authoritative entry point for all logging events. It achieves this through several key mechanisms:
1.  **JVM Integration:** During static initialization, it forcibly replaces the default `java.util.logging.LogManager` with a custom `HytaleLogManager`. This is a critical, system-wide hook that allows it to intercept logging calls from third-party libraries that use the standard Java logging framework.
2.  **Stream Redirection:** It provides a mechanism to capture the global `System.out` and `System.err` streams, redirecting their output into the Hytale logging pipeline. This ensures that even simple `println` calls are processed by the engine's logging infrastructure.
3.  **Backend Abstraction:** The HytaleLogger class itself is a lightweight frontend. All log processing, formatting, and dispatching is delegated to a `HytaleLoggerBackend`. This backend is responsible for routing log records to various sinks, such as the `HytaleConsole`, `HytaleFileHandler`, and external error reporting services like Sentry.
4.  **Cached Instances:** To ensure performance and instance uniqueness, loggers are managed in a static, concurrent cache. Requesting a logger for a given name will always return the same instance.

This component must be one of the first classes initialized during application startup to guarantee its control over the JVM's logging and output subsystems.

### Lifecycle & Ownership
- **Creation:** The static components of HytaleLogger, including the root logger and the logger cache, are created once by the JVM classloader during the static initialization phase. This process is triggered the very first time any part of the HytaleLogger class is accessed. It is designed to be one of the earliest initializations in the application's lifecycle. Individual logger instances are created on-demand and cached by the static `get` and `forEnclosingClass` factory methods.
- **Scope:** The HytaleLogger system is application-scoped. It persists for the entire lifetime of the JVM process.
- **Destruction:** State is managed by the static `CACHE` and is only reclaimed upon JVM shutdown. There is no explicit teardown or `close` method, as it is a core, persistent service.

## Internal State & Concurrency
- **State:** The primary shared state is the static `CACHE`, a `ConcurrentHashMap` that maps logger names to their corresponding HytaleLogger instances. This state is mutable, as new loggers are added on-demand. The configuration state for a specific logger, such as its `Level`, is managed within its associated `HytaleLoggerBackend` instance, not in the HytaleLogger frontend itself.
- **Thread Safety:** This class is thread-safe. The logger cache uses `ConcurrentHashMap` to guarantee safe, atomic creation and retrieval of logger instances from multiple threads. The logging methods themselves are designed for high-concurrency scenarios, delegating work to the backend which is responsible for its own thread safety, typically through synchronized or queue-based appenders.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init() | static void | O(1) | Performs secondary initialization, activating configured log handlers like file and console sinks. Must be called early in the startup sequence. |
| replaceStd() | static void | O(1) | Redirects the JVM's `System.out` and `System.err` streams to this logging system. **Warning:** This is a global, irreversible action for the process. |
| get(String name) | static HytaleLogger | O(1) amortized | Factory method to retrieve a cached or new logger instance for the given name. This is the preferred method for creating loggers for specific subsystems. |
| forEnclosingClass() | static HytaleLogger | O(k) | Convenience factory to get a logger for the calling class. Incurs minor performance overhead due to stack walking (k = stack depth). |
| at(Level level) | Api | O(1) | The primary entry point for creating a fluent log statement. Returns a no-op implementation if the level is not enabled, preventing argument evaluation. |
| setLevel(Level level) | void | O(1) | Sets the minimum log level for this logger instance and its children, delegating the change to the backend. |
| setSentryClient(IScopes scope) | void | O(1) | Binds a Sentry client to this logger's backend, enabling remote error and event reporting for logs processed by this instance. |

## Integration Patterns

### Standard Usage
The most common pattern is to declare a static final logger instance in each class. This is efficient as the logger is retrieved only once per class.

```java
// Standard retrieval at the class level for optimal performance.
private static final HytaleLogger logger = HytaleLogger.forEnclosingClass();

public void processData(Object data) {
    if (data == null) {
        // The fluent API prevents string formatting if the level is disabled.
        logger.at(Level.WARNING).log("Received null data packet.");
        return;
    }
    logger.at(Level.FINE).log("Processing data packet of type: %s", data.getClass().getName());
    // ... processing logic
}
```

### Anti-Patterns (Do NOT do this)
- **Late Initialization:** Calling `init()` or `replaceStd()` after other parts of the application have started can result in lost log messages or unpredictable console output from third-party libraries. These methods must be invoked from the primary application entry point.
- **Per-Method Logger Retrieval:** Do not call `HytaleLogger.forEnclosingClass()` inside a method that is called frequently. The stack walking required to find the caller's class name introduces unnecessary performance overhead.
- **String Concatenation in Log Calls:** Avoid manual string concatenation. This defeats the performance benefits of the fluent API, as the string is constructed even if the log level is disabled.
    - **BAD:** `logger.at(Level.FINE).log("Value is " + expensiveOperation());`
    - **GOOD:** `logger.at(Level.FINE).log("Value is %s", expensiveOperation());`

## Data Pipeline
The flow of a single log event is designed for high performance and flexibility, moving from the high-level API to the low-level sinks.

> Flow:
> Developer Call (`logger.at(...).log(...)`) -> **HytaleLogger** (Fluent API check) -> `HytaleLoggerBackend` (Log record creation) -> Dispatcher -> Multiple Sinks (`HytaleConsole`, `HytaleFileHandler`, Sentry Client)

