---
description: Architectural reference for LoggerUtil
---

# LoggerUtil

**Package:** com.hypixel.hytale.builtin.hytalegenerator
**Type:** Utility

## Definition
```java
// Signature
public class LoggerUtil {
```

## Architecture & Concepts
LoggerUtil is a stateless, static utility class designed to centralize and standardize logging operations within the Hytale world generation subsystem. Its primary architectural function is to act as a facade over the standard Java Util Logging framework, ensuring that all exceptions originating from the generator are formatted and reported in a consistent manner.

By providing a single point of entry for exception logging, this class decouples generator components from the underlying logging implementation. It enforces a specific format that includes a high-level context description and a full stack trace, which is critical for debugging complex, multi-stage generation processes. It is a foundational component, not intended for business logic, but for low-level error reporting infrastructure.

### Lifecycle & Ownership
- **Creation:** As a static utility class, LoggerUtil is never instantiated. The class is loaded by the JVM ClassLoader, and its static members become available.
- **Scope:** The class and its static methods are available for the entire application lifetime.
- **Destruction:** The class is unloaded when the JVM terminates. There are no instances to manage or destroy.

## Internal State & Concurrency
- **State:** LoggerUtil is completely stateless. It maintains no internal fields or caches. The HYTALE_GENERATOR_NAME field is a public, static, final constant resolved at compile time.
- **Thread Safety:** This class is inherently thread-safe. Its methods operate exclusively on their arguments and do not modify any shared state. The underlying Java Logger instances obtained via Logger.getLogger are themselves thread-safe, making concurrent calls to LoggerUtil methods safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getLogger() | Logger | O(1) | Retrieves the shared Logger instance for the HytaleGenerator subsystem. |
| logException(context, e) | void | O(N) | Formats and logs an exception to the default HytaleGenerator logger. Complexity is proportional to the stack trace depth (N). |
| logException(context, e, logger) | void | O(N) | Formats and logs an exception to a user-specified logger instance. |

## Integration Patterns

### Standard Usage
The primary use case is to wrap potentially failing operations within a try-catch block and report exceptions through this utility. This ensures consistent error visibility in server logs.

```java
// How a developer should normally use this
try {
    // A complex operation within the world generator
    terrainFeature.applyToChunk(chunk);
} catch (Exception e) {
    // Provide specific context for easier debugging
    LoggerUtil.logException("applying terrain feature " + terrainFeature.getName(), e);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new LoggerUtil()`. This provides no value as all methods are static and it pollutes the heap unnecessarily.
- **Vague Context:** Calling `logException("Error", e)` is an anti-pattern. The context description is the most valuable piece of information for diagnostics and must be specific to the operation that failed.
- **Logging and Swallowing:** Do not use this utility to log an exception and then continue execution as if no error occurred, unless the error is explicitly recoverable. This can lead to silent failures and corrupted world state.

## Data Pipeline
LoggerUtil acts as a simple transformation and routing step in the engine's error handling pipeline. It does not queue or batch data; calls are processed synchronously.

> Flow:
> `Throwable` Object -> **LoggerUtil.logException()** -> String Formatting via ExceptionUtil -> `java.util.logging.Logger` -> Configured Log Appender (File, Console)

