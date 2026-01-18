---
description: Architectural reference for HytaleLogFormatter
---

# HytaleLogFormatter

**Package:** com.hypixel.hytale.logger.backend
**Type:** Transient

## Definition
```java
// Signature
public class HytaleLogFormatter extends Formatter {
```

## Architecture & Concepts
The HytaleLogFormatter is a specialized component within the engine's logging subsystem. It operates as a pluggable strategy for the standard Java Logging (JUL) framework, inheriting from `java.util.logging.Formatter`.

Its primary architectural role is to transform abstract `LogRecord` data structures into a standardized, human-readable string representation. This class is solely responsible for the final visual presentation of logs, including timestamps, log levels, module names, and color-coding.

Key architectural features include:
*   **Adaptive Layout:** It implements a dynamic column-width algorithm for the module name field. This ensures that log messages from different sources are vertically aligned in the output, significantly improving readability without requiring manual configuration.
*   **Conditional ANSI Coloring:** The formatter can be configured at runtime to enable or disable ANSI escape codes for colored output. This is managed via a `BooleanSupplier` dependency, allowing the logging backend to adapt to the capabilities of the output console (e.g., IDE console vs. a plain text file).
*   **Raw Record Passthrough:** It contains a special-case handler for `HytaleLoggerBackend.RawLogRecord`. This allows certain parts of the engine to bypass the standard formatting and output pre-formatted or raw text directly to the log stream, providing an escape hatch for custom logging needs.

This class is a foundational piece of the developer-facing diagnostic and debugging experience.

## Lifecycle & Ownership
- **Creation:** An instance of HytaleLogFormatter is not intended for direct creation by application code. It is instantiated and configured by the logging bootstrap system, typically within the `HytaleLoggerBackend` or a similar central configuration point. It is then assigned to a `java.util.logging.Handler`.
- **Scope:** The lifecycle of a HytaleLogFormatter instance is tightly coupled to the `Handler` that owns it. It persists as long as its parent `Handler` is active, which is generally for the entire application session.
- **Destruction:** The object is eligible for garbage collection when the logging system is shut down and the associated `Handler` is closed and dereferenced. There is no explicit `destroy` or `close` method.

## Internal State & Concurrency
- **State:** HytaleLogFormatter is a **stateful, mutable** component. It maintains internal state via the `maxModuleName` and `shorterCount` fields. These fields are modified during calls to the `format` method to track and adjust the padding for logger names. This stateful nature is essential for its adaptive layout feature.

- **Thread Safety:** This class is **not thread-safe**. The read-modify-write operations on the `maxModuleName` and `shorterCount` fields are not synchronized. If a single instance were to be used by multiple `Handler` objects operating concurrently, it would result in a race condition, leading to corrupted log alignment.

    **Warning:** The standard Java Logging framework typically mitigates this risk by synchronizing the `publish` method on its `Handler` implementations, which effectively serializes calls to the attached formatter. However, this is a characteristic of the environment, not a guarantee provided by this class. Relying on this class in any multi-threaded context without external locking is unsafe.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| format(LogRecord record) | String | O(L) | The primary entry point. Transforms a LogRecord into a formatted string. Modifies internal state for column alignment. L is the length of the log message. |
| stripAnsi(String message) | static String | O(M) | A static utility method to remove all ANSI escape codes from a string. M is the length of the input string. |

## Integration Patterns

### Standard Usage
This class is not designed to be used directly. It is a component configured by the core logging system and attached to a log handler. The conceptual setup is as follows.

```java
// Conceptual example of how the logging backend configures this class.
// Do not replicate this in application code.

// 1. A supplier is created to determine if the console supports color.
BooleanSupplier ansiEnabled = () -> System.console() != null;

// 2. The formatter is instantiated with its dependency.
HytaleLogFormatter formatter = new HytaleLogFormatter(ansiEnabled);

// 3. The formatter is attached to a handler, which directs output.
java.util.logging.Handler consoleHandler = new java.util.logging.ConsoleHandler();
consoleHandler.setFormatter(formatter);

// 4. The handler is registered with a logger.
java.util.logging.Logger.getLogger("").addHandler(consoleHandler);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation for Logging:** Never create an instance of this class just to format a single message. It is a long-lived component of the logging framework.
- **Shared Instances:** Do not create a single HytaleLogFormatter and share it across multiple, unsynchronized log handlers. The stateful column-sizing mechanism will fail due to race conditions. Each handler that may operate on a separate thread should have its own formatter instance.

## Data Pipeline
The HytaleLogFormatter sits in the middle of the log processing pipeline, acting as the final transformation stage before output.

> Flow:
> Engine Code calls `Logger.info()` -> `LogRecord` created -> `java.util.logging.Handler` receives record -> Handler invokes **HytaleLogFormatter.format()** -> Formatted `String` returned -> Handler writes `String` to Console/File Stream

