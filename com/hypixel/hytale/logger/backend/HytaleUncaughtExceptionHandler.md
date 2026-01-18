---
description: Architectural reference for HytaleUncaughtExceptionHandler
---

# HytaleUncaughtExceptionHandler

**Package:** com.hypixel.hytale.logger.backend
**Type:** Singleton

## Definition
```java
// Signature
public class HytaleUncachedExceptionHandler implements Thread.UncaughtExceptionHandler {
```

## Architecture & Concepts
The HytaleUncaughtExceptionHandler is a foundational component of the engine's stability and diagnostics framework. It functions as a global, last-resort safety net for the entire Java Virtual Machine (JVM) process, ensuring that no critical failure goes unrecorded.

Its primary architectural role is to intercept any Throwable that propagates up a thread's call stack without being caught. By registering itself as the default handler for all threads—including the common ForkJoinPool used for asynchronous tasks—it centralizes error handling at the lowest level. This prevents unhandled exceptions from being silently discarded or simply printed to a standard error stream, which could be lost. Instead, all such terminal events are funneled directly into the HytaleLogger system, providing a unified and reliable source for crash analysis.

This class is critical for creating robust server and client applications, as it guarantees that even unexpected failures in third-party libraries or complex multithreaded scenarios are captured and logged correctly.

### Lifecycle & Ownership
- **Creation:** The singleton INSTANCE is instantiated eagerly by the JVM during static initialization of the class. This occurs very early in the application bootstrap sequence, ensuring the handler is available before any application code executes.
- **Scope:** This object is a process-global singleton with a static lifetime. It persists from the moment its class is loaded until the JVM process is terminated.
- **Destruction:** The object is not explicitly destroyed. It is reclaimed by the operating system along with the rest of the process heap upon JVM shutdown.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no instance fields and its behavior is determined solely by the arguments provided to its methods. Its purpose is purely functional.
- **Thread Safety:** The HytaleUncaughtExceptionHandler is inherently **thread-safe**. The uncaughtException method is designed to be called by any thread at any time without external synchronization. Its stateless nature eliminates the possibility of race conditions or data corruption.

## API Surface
The public contract is minimal, designed for one-time setup rather than continuous interaction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup() | static void | O(1) | Registers the singleton instance as the default uncaught exception handler for the entire JVM. This is the primary entry point. |
| uncaughtException(Thread t, Throwable e) | void | O(1) | The callback method invoked by the JVM. Captures the exception and routes it to the HytaleLogger. **WARNING:** Do not call this method directly. |

## Integration Patterns

### Standard Usage
The handler must be configured once at the earliest possible point in the application's entry point. This ensures all subsequent threads, including the main thread, are covered.

```java
// Correct usage: Call once during application startup.
// This is typically done in the main method or an equivalent bootstrap class.

public static void main(String[] args) {
    HytaleUncaughtExceptionHandler.setup();

    // ... proceed with application initialization
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create a new instance with `new HytaleUncaughtExceptionHandler()`. The system is designed around a single, static instance. Creating others serves no purpose and can lead to confusion.
- **Manual Invocation:** Do not call the `uncaughtException` method from your own code. It is a callback intended exclusively for the JVM runtime. Manually invoking it will create misleading log entries and subvert the natural exception handling process.
- **Delayed Setup:** Calling `setup` late in the initialization sequence is dangerous. Any threads spawned before the call will not be covered by this handler, creating a gap in diagnostic coverage.

## Data Pipeline
This class acts as a terminal sink in the exception propagation pipeline, converting a JVM-level event into a structured log entry.

> Flow:
> Unhandled Throwable in any Thread -> JVM Runtime -> **HytaleUncaughtExceptionHandler.uncaughtException** -> HytaleLogger API -> Log Appender (File, Console, Network)

