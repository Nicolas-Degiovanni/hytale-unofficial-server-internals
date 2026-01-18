---
description: Architectural reference for SkipSentryException
---

# SkipSentryException

**Package:** com.hypixel.hytale.logger.sentry
**Type:** Signal Exception

## Definition
```java
// Signature
public class SkipSentryException extends RuntimeException {
```

## Architecture & Concepts
SkipSentryException is a specialized exception class that functions as a **control signal** for the engine's error reporting pipeline. Its primary purpose is not to represent a specific programming error, but rather to instruct the global exception handler to suppress reporting a given throwable to the Sentry service.

In a large-scale application, certain exceptions are expected and do not represent a bug or a critical failure. Examples include user-cancelled downloads, intentional connection closures, or recoverable asset loading failures. Reporting these events to a service like Sentry creates significant noise, making it difficult to identify actionable issues.

This class provides a mechanism to tag an exception chain as "benign" or "known". The central error handling logic uses the static utility method, hasSkipSentry, to inspect an exception's cause chain. If an instance of SkipSentryException is found anywhere in the chain, the exception is not sent to Sentry, effectively filtering it from remote error monitoring.

### Lifecycle & Ownership
- **Creation:** Instantiated dynamically by application code, typically within a `catch` block, to wrap an underlying exception that should not be reported. It can also be thrown directly to signal a non-reportable event.
- **Scope:** Transient and extremely short-lived. An instance exists only for the duration of its propagation up the call stack until it is intercepted and processed by the centralized error handling system.
- **Destruction:** The object is eligible for garbage collection immediately after the error handler has inspected it. It holds no resources and has no explicit cleanup requirements.

## Internal State & Concurrency
- **State:** Effectively immutable. While inheriting from RuntimeException, its state (message and cause) is set at construction and is not intended to be modified. It is a stateless signal.
- **Thread Safety:** This class is inherently thread-safe. Exception instances are confined to the thread in which they are thrown. The static `hasSkipSentry` method is a pure function that operates solely on its input, making it safe for concurrent invocation.

## API Surface
The primary public contract is the static `hasSkipSentry` method, which is the integration point for the error reporting system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hasSkipSentry(Throwable thrown) | static boolean | O(N) | Traverses the entire exception cause chain, where N is the depth of causes. Returns true if any throwable in the chain is an instance of SkipSentryException. This is the core filtering logic. |

## Integration Patterns

### Standard Usage
The intended consumer of this class is the global error handler or a Sentry `BeforeSendCallback` implementation. The handler catches all unhandled exceptions and uses this class to decide whether to report them.

```java
// Example from a central Sentry integration point
public class SentryErrorFilter {

    public void processError(Throwable error) {
        // The core integration pattern: check before reporting.
        if (SkipSentryException.hasSkipSentry(error)) {
            // Log locally for debugging but do not send to Sentry.
            System.err.println("Suppressed Sentry report for a known exception: " + error.getMessage());
        } else {
            // This is a legitimate, unknown error. Report it.
            Sentry.captureException(error);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Empty Catch Blocks:** Do not catch a SkipSentryException and swallow it. Its purpose is to signal the *global* handler. Allowing it to propagate is the correct pattern.
    ```java
    // BAD: This hides the exception from all logging.
    try {
        mightThrowBenignError();
    } catch (SkipSentryException e) {
        // The exception is now lost.
    }
    ```
- **Overuse:** Do not wrap every possible exception. This mechanism should be reserved for errors that are truly expected under normal operation and provide no value to remote monitoring. Overuse can mask real, underlying issues.

## Data Pipeline
SkipSentryException acts as a conditional gate in the error processing pipeline. It does not transform data but rather directs its flow.

> Flow:
> Application Logic Throws Exception -> Call Stack Unwinds -> Global Exception Handler -> **SkipSentryException.hasSkipSentry()** -> [If True] -> Local Log / No-Op
>
> **...or...**
>
> Global Exception Handler -> **SkipSentryException.hasSkipSentry()** -> [If False] -> Sentry SDK -> Network -> Sentry Service

