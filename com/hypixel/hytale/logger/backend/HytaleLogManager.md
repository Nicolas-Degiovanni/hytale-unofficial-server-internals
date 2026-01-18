---
description: Architectural reference for HytaleLogManager
---

# HytaleLogManager

**Package:** com.hypixel.hytale.logger.backend
**Type:** Singleton

## Definition
```java
// Signature
public class HytaleLogManager extends LogManager {
```

## Architecture & Concepts
HytaleLogManager is a foundational component of the engine's logging subsystem. It functions as a high-level adapter, bridging the standard Java Util Logging (JUL) framework with Hytale's specialized logging backend. By extending the JVM's own LogManager, it intercepts all standard logging requests at their source.

Its primary architectural purpose is to ensure that all logging activity, including that from third-party libraries that rely on JUL, is captured and routed through Hytale's custom infrastructure. This allows for unified log formatting, filtering, and output to destinations like the in-game console (HytaleConsole) and log files (HytaleFileHandler).

A key design feature is the deliberate override of the standard reset method. This provides stability by preventing any component from globally wiping the engine's logging configuration, a common issue in complex applications. The actual shutdown and reset logic is exposed via a controlled, static method, resetFinally, intended for use only during the application's final shutdown sequence.

## Lifecycle & Ownership
- **Creation:** HytaleLogManager is instantiated automatically by the Java Virtual Machine during its startup sequence. This is configured by setting the `java.util.logging.manager` system property to point to this class. Application code should **never** instantiate this class directly.
- **Scope:** As a JVM-level singleton, its lifecycle is tied to the application process itself. It is created once at startup and persists until the JVM shuts down.
- **Destruction:** The manager is effectively destroyed upon JVM exit. The static method resetFinally serves as the explicit shutdown hook for the logging system. It must be called to ensure all log handlers are properly flushed and closed, preventing data loss. This is typically invoked from a central application shutdown handler.

## Internal State & Concurrency
- **State:** The class itself is largely stateless, with its primary state being the static singleton reference, instance. The true state, which is the registry of named loggers, is managed by the parent `java.util.logging.LogManager` class.
- **Thread Safety:** The class relies on the thread-safety guarantees of the parent `java.util.logging.LogManager` for managing the logger cache. The public methods are safe for concurrent access. The static instance field is assigned in the constructor and is not volatile; however, since the JVM manages its creation in a controlled, single-threaded manner during startup, this does not pose a practical concurrency risk.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getLogger(String name) | Logger | O(log N) | Intercepts requests for a logger. Returns a JUL-compatible Logger that wraps and delegates all calls to the HytaleLoggerBackend. |
| reset() | void | O(1) | Overridden to be a no-op. Prevents external code from destructively resetting the engine's logging configuration. |
| resetFinally() | void | O(1) | **Critical Shutdown Method.** Statically orchestrates the graceful shutdown of all custom Hytale log handlers before performing a final reset. |

## Integration Patterns

### Standard Usage
Developers do not interact with HytaleLogManager directly. Its integration is transparent. The standard Java logging API is used, and HytaleLogManager automatically routes the calls.

```java
// This standard Java API call is automatically intercepted
// by HytaleLogManager and routed to the Hytale backend.
import java.util.logging.Logger;

Logger logger = Logger.getLogger("com.hytale.MySystem");
logger.info("System has started successfully.");
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new HytaleLogManager()`. The JVM is solely responsible for its creation via system properties. Direct instantiation will violate the singleton contract and lead to unpredictable logging behavior.
- **Bypassing Shutdown:** Failure to call the static `HytaleLogManager.resetFinally()` method during application shutdown will result in unflushed log buffers and potential loss of the last log messages.
- **Calling reset:** Calling the instance method `reset()` will have no effect and indicates a misunderstanding of the class's design. Use the static `resetFinally()` for shutdown.

## Data Pipeline
The flow of a log message is orchestrated by this manager, ensuring all output is standardized and controlled.

> Flow:
> Standard `Logger.getLogger()` call -> JVM directs to **HytaleLogManager** -> `getLogger()` returns a `HytaleJdkLogger` wrapper -> Application calls `logger.info()` -> `HytaleJdkLogger` delegates to `HytaleLoggerBackend` -> Backend dispatches LogRecord to `HytaleConsole` and `HytaleFileHandler`

