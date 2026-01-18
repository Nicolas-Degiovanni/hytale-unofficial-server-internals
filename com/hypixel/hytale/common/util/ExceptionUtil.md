---
description: Architectural reference for ExceptionUtil
---

# ExceptionUtil

**Package:** com.hypixel.hytale.common.util
**Type:** Utility

## Definition
```java
// Signature
public class ExceptionUtil {
```

## Architecture & Concepts
ExceptionUtil is a stateless, engine-wide utility class designed to provide a centralized and consistent approach to processing `Throwable` objects. It is a foundational component of the Hytale error handling and diagnostics pipeline.

Its primary architectural role is to decouple the *format* of an error report from the *handling* of the error itself. Systems across the client and server—from networking to asset loading—do not need to implement their own logic for traversing exception causes or serializing stack traces. Instead, they delegate this responsibility to ExceptionUtil, ensuring that all diagnostic output, whether logged to a file or displayed to a user, follows a uniform structure. This standardization is critical for efficient debugging and automated crash report analysis.

## Lifecycle & Ownership
- **Creation:** Not applicable. As a pure utility class with only static methods, ExceptionUtil is never instantiated.
- **Scope:** Application-wide. The class and its methods are available globally as soon as the class is loaded by the JVM ClassLoader.
- **Destruction:** Not applicable. The class is unloaded upon JVM shutdown.

## Internal State & Concurrency
- **State:** Stateless. This class contains no instance or static fields, and its methods operate exclusively on the arguments provided. Each call is an independent, pure function.
- **Thread Safety:** Fully thread-safe. Due to its stateless nature, all methods can be invoked concurrently from any thread without requiring external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| combineMessages(Throwable, String) | String | O(N) | Traverses the exception cause chain (depth N) and concatenates all non-null messages. Protects against circular cause references. |
| toStringWithStack(Throwable) | String | O(S) | Serializes the entire throwable, including its full stack trace of size S, into a single string. This is a potentially expensive operation. |

## Integration Patterns

### Standard Usage
This utility should be invoked within catch blocks to prepare exceptions for logging or reporting. The choice between methods depends on the desired level of detail.

```java
// Standard pattern for logging a detailed error report
try {
    // An operation that can throw a nested exception
} catch (Throwable t) {
    // For detailed debugging and crash reports, capture the full stack trace.
    String fullReport = ExceptionUtil.toStringWithStack(t);
    CrashReporter.submit(fullReport);

    // For less verbose logging, summarize the cause chain.
    String summary = ExceptionUtil.combineMessages(t, " caused by: ");
    Logger.error("A critical error occurred: " + summary);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never attempt to create an instance via `new ExceptionUtil()`. All methods are static and should be accessed directly on the class.
- **Performance-Sensitive Calls:** Avoid calling `toStringWithStack` within tight, performance-critical loops such as the main game update or rendering thread. The overhead of capturing the stack trace can introduce significant stutter. If an exception occurs in such a loop, log a minimal message and defer detailed processing to a separate diagnostics thread if possible.
- **Ignoring Return Value:** The methods in this class do not have side effects; they return a new string. Calling a method without assigning or using its result is a no-op and serves no purpose.

## Data Pipeline
ExceptionUtil acts as a transformation component in the engine's error handling data flow. It converts a raw, structured object into a human-readable, unstructured string format suitable for output.

> Flow:
> Raw Throwable Object -> **ExceptionUtil** -> Formatted String -> Logger / Crash Reporter / UI Error Message

