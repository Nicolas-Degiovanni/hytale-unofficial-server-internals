---
description: Architectural reference for HytaleSentryHandler
---

# HytaleSentryHandler

**Package:** com.hypixel.hytale.logger.sentry
**Type:** Handler

## Definition
```java
// Signature
public class HytaleSentryHandler extends Handler {
```

## Architecture & Concepts
The HytaleSentryHandler acts as a critical bridge between the standard Java Util Logging (JUL) framework and the Sentry error reporting service. By extending the `java.util.logging.Handler` class, it integrates directly into the Java logging pipeline, allowing it to intercept all log messages processed by the loggers it is attached to.

Its primary architectural function is to translate a standard `LogRecord` object into three distinct types of Sentry data, based on configurable severity levels:

1.  **Sentry Events:** For high-severity logs (e.g., SEVERE), the handler creates a full Sentry event. This represents a significant, reportable error that will appear as a new issue in the Sentry dashboard.
2.  **Sentry Breadcrumbs:** For moderate-severity logs (e.g., INFO), the handler creates a breadcrumb. Breadcrumbs are a chronological trail of application events that provide context for a future Sentry event, helping developers trace the steps that led to an error.
3.  **Sentry Logs:** For lower-severity logs, the handler can forward the message to Sentry's dedicated logging infrastructure, which is distinct from its error eventing system.

This multi-level processing allows for a nuanced approach to diagnostics: capturing detailed context for all operations while only escalating critical failures into actionable Sentry issues.

### Lifecycle & Ownership
- **Creation:** The HytaleSentryHandler is designed to be instantiated once during the application's bootstrap phase. It is typically configured programmatically or declaratively via a `logging.properties` file and then added to a root or package-specific `Logger`.
- **Scope:** Session-scoped. The handler's lifecycle is bound to the `Logger` it is registered with, persisting for the entire duration of the application's runtime.
- **Destruction:** The `close` method is invoked by the `LogManager` during a graceful application shutdown. This is a critical step that calls `Sentry.close`, which is responsible for flushing any buffered events to the Sentry service. Failure to trigger this cleanup will result in data loss.

## Internal State & Concurrency
- **State:** Mutable. The handler maintains its configuration as internal state, including the minimum `Level` required to trigger events, breadcrumbs, and logs. This configuration is typically loaded from the `LogManager` at startup but can be modified at runtime via public setters.
- **Thread Safety:** The core `publish` method is thread-safe, as it delegates state modification and network I/O to the underlying Sentry SDK, which is designed for high-concurrency environments.

    **Warning:** While the `publish` method is safe, concurrently modifying the handler's configuration (e.g., calling `setMinimumEventLevel` from multiple threads) is not an atomic operation and is not recommended. All configuration should be performed in a single-threaded context during application initialization.

## API Surface
The public API is primarily for configuration and lifecycle management by the logging framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| publish(LogRecord record) | void | O(1) | Core entry point called by the JUL framework. Translates a LogRecord into Sentry data and dispatches it asynchronously. |
| close() | void | O(N) | Flushes any buffered events to the Sentry service. N is the number of events in the buffer. This can block. |
| setMinimumEventLevel(Level) | void | O(1) | Sets the minimum JUL level required to capture a full Sentry event. |
| setMinimumBreadcrumbLevel(Level) | void | O(1) | Sets the minimum JUL level required to record a Sentry breadcrumb. |

## Integration Patterns

### Standard Usage
The handler should be configured and added to a logger during application startup. It is not intended for ad-hoc instantiation.

```java
// Example: Programmatic setup during application bootstrap
import java.util.logging.Logger;
import io.sentry.Sentry;

// Assuming Sentry has already been initialized with Sentry.init(...)
Logger rootLogger = Logger.getLogger("");

// Remove default handlers to avoid duplicate console output if desired
for (Handler handler : rootLogger.getHandlers()) {
    rootLogger.removeHandler(handler);
}

// Create and add the Sentry handler
HytaleSentryHandler sentryHandler = new HytaleSentryHandler(Sentry.getCurrentScope());
sentryHandler.setMinimumEventLevel(Level.SEVERE);
sentryHandler.setMinimumBreadcrumbLevel(Level.INFO);

rootLogger.addHandler(sentryHandler);
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Never call the `publish` method directly. The Java Util Logging framework is solely responsible for invoking this method. Manual calls will bypass the framework's filtering and context management.
- **Missing Shutdown Hook:** Failure to ensure `LogManager.getLogManager().reset()` or a similar mechanism calls `close` on application exit will lead to the loss of any buffered Sentry events that have not yet been transmitted.
- **Filter Removal:** The handler self-installs a `DropSentryFilter` to prevent logs originating from the Sentry client itself from being processed. Removing this filter will create an infinite logging loop, causing a `StackOverflowError`.

## Data Pipeline
The handler acts as a transformation and dispatch node within the application's logging and error reporting pipeline.

> Flow:
> Application Code (`Logger.severe`) -> `java.util.logging.LogRecord` -> JUL Framework -> **HytaleSentryHandler.publish** -> Sentry SDK (Event/Breadcrumb) -> Sentry Transport (Network Queue) -> Sentry Service

