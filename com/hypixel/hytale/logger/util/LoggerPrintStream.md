---
description: Architectural reference for LoggerPrintStream
---

# LoggerPrintStream

**Package:** com.hypixel.hytale.logger.util
**Type:** Stateful Utility

## Definition
```java
// Signature
public class LoggerPrintStream extends PrintStream {
```

## Architecture & Concepts
LoggerPrintStream is an adapter that bridges standard Java I/O streams to the Hytale structured logging system. Its primary role is to capture byte-stream output, typically from `System.out` and `System.err`, and redirect it as discrete log messages to a designated HytaleLogger instance.

This component is critical for ensuring that all output generated within the JVM, including from third-party libraries or native code that writes to standard output, is captured and processed by the engine's centralized logging pipeline. It implements the Adapter Pattern, translating the low-level, line-oriented `PrintStream` API into the structured, level-aware API of HytaleLogger.

The core mechanism operates by buffering incoming bytes until a line terminator (CR, LF, or CRLF) is detected. Upon detection, the entire buffered line is flushed as a single log entry at a pre-configured logging level.

**WARNING:** This class is designed for system-level stream redirection. It is not intended for general application logging; use HytaleLogger directly for that purpose to leverage structured logging features.

### Lifecycle & Ownership
- **Creation:** Instantiated during the engine's bootstrap phase, typically by a central logging service or manager. Two distinct instances are usually created: one for `System.out` and another for `System.err`, each configured with an appropriate logging level (e.g., INFO and SEVERE).
- **Scope:** Session-scoped. Once installed via `System.setOut` or `System.setErr`, an instance persists for the entire lifetime of the application.
- **Destruction:** There is no explicit destruction method. The object is eligible for garbage collection upon application shutdown when the JVM terminates.

## Internal State & Concurrency
- **State:** LoggerPrintStream is stateful. It maintains an internal `ByteArrayOutputStream` to buffer characters that form a single line. It also tracks the last character received to correctly handle CRLF line endings without generating duplicate log entries. This state is mutable and essential to its line-parsing logic.
- **Thread Safety:** This class is thread-safe. The parent `java.io.PrintStream` class synchronizes its public `write` methods. As LoggerPrintStream overrides these methods, all internal state modifications (such as writing to the buffer and flushing to the logger) are performed within a synchronized context, preventing race conditions from concurrent writes to the stream.

## API Surface
The public API is inherited from `java.io.PrintStream`. The methods below are the core implementation overrides.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| write(int b) | void | Amortized O(1) | Buffers a single byte. If the byte is a line terminator, flushes the entire buffer to the HytaleLogger. |
| write(byte[] buf, int off, int len) | void | O(N) | Writes a portion of a byte array by delegating to the single-byte write method. N is the length. |
| getLogger() | HytaleLogger | O(1) | Returns the HytaleLogger instance this stream forwards messages to. |
| getLevel() | Level | O(1) | Returns the `java.util.logging.Level` used for all messages logged by this stream. |

## Integration Patterns

### Standard Usage
The sole intended use case is to replace the standard system output and error streams during application initialization.

```java
// In a logging initialization service
HytaleLogger systemOutLogger = ...;
HytaleLogger systemErrLogger = ...;

// Redirect System.out to log at INFO level
PrintStream outStream = new LoggerPrintStream(systemOutLogger, Level.INFO);
System.setOut(outStream);

// Redirect System.err to log at SEVERE level
PrintStream errStream = new LoggerPrintStream(systemErrLogger, Level.SEVERE);
System.setErr(errStream);

// All subsequent calls will be captured
System.out.println("This is now a structured log message.");
System.err.println("This is now a severe error log message.");
```

### Anti-Patterns (Do NOT do this)
- **Direct Usage:** Do not instantiate and use this class as a general-purpose stream. It is less performant than writing directly to a file or console and bypasses the benefits of structured logging. Always prefer using the HytaleLogger API directly.
- **Incorrect Level Configuration:** Assigning an inappropriate level can lead to miscategorized logs. For example, redirecting `System.err` to a logger with `Level.FINE` would mask critical errors from standard views.

## Data Pipeline
LoggerPrintStream acts as an ingress point, converting unstructured byte streams into the engine's structured log event pipeline.

> Flow:
> `System.out.println()` -> JVM Standard Stream -> **LoggerPrintStream.write()** -> Internal Byte Buffer -> `HytaleLogger.at(level).log()` -> Log Appenders (Console, File, Network)

