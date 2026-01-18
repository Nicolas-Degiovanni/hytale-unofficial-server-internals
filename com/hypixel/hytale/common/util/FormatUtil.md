---
description: Architectural reference for FormatUtil
---

# FormatUtil

**Package:** com.hypixel.hytale.common.util
**Type:** Utility

## Definition
```java
// Signature
public class FormatUtil {
```

## Architecture & Concepts
FormatUtil is a stateless, static utility class that serves as the centralized service for converting raw data types into human-readable strings. Its primary architectural role is to decouple formatting logic from the systems that produce or consume the data, such as performance monitoring, logging, and user interface rendering.

By providing a single, consistent implementation for formatting common data types like time durations, byte sizes, and performance metrics, it enforces a uniform presentation style across the entire application. This prevents code duplication and ensures that developers do not need to re-implement complex or error-prone formatting logic. It is a foundational, low-level component with no dependencies on higher-level game systems.

## Lifecycle & Ownership
- **Creation:** The FormatUtil class is never instantiated. As a class containing only static members, its methods are accessed directly on the class object itself. The class is loaded into the JVM by the ClassLoader when it is first referenced by another component.
- **Scope:** Application-wide. Once loaded, its static methods are available for the entire lifetime of the JVM process.
- **Destruction:** The class is unloaded from memory when the application terminates and the JVM shuts down. There are no instances to manage or clean up.

## Internal State & Concurrency
- **State:** FormatUtil is **stateless**. All internal fields are declared as static final, acting as immutable constants. Data structures like the timeUnitToShortString map are initialized once in a static initializer block and are not modified at runtime.
- **Thread Safety:** This class is inherently **thread-safe**. All methods are pure functions that operate exclusively on their input arguments without modifying any shared state. They can be safely invoked from any thread simultaneously without requiring external synchronization or locks.

## API Surface
The public API consists entirely of static methods designed for specific formatting tasks.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| bytesToString(long bytes) | String | O(1) | Converts a raw byte count into a human-readable string with appropriate units (e.g., "1.5 KiB", "2.0 MB"). |
| nanosToString(long nanos) | String | O(1) | Formats a nanosecond duration into a detailed, multi-part string (e.g., "1min 30s 500ms"). |
| simpleTimeUnitFormat(...) | String | O(1) | Provides multiple overloads to format a time value into a compact string with its largest significant unit (e.g., "12.5ms"). |
| simpleFormat(Metric metric, ...) | String | O(1) | Formats a Metric object into a summary string showing "average +/- range". |
| addNumberSuffix(int i) | String | O(1) | Appends an ordinal suffix to an integer (e.g., 1 becomes "1st", 2 becomes "2nd"). |

## Integration Patterns

### Standard Usage
Methods should always be invoked statically. This utility is commonly used in logging, debug overlays, and any UI component that displays performance or system data.

```java
// Example: Formatting a file size for a log message
long fileSize = 10485760; // 10 MiB
String formattedSize = FormatUtil.bytesToString(fileSize);
LOGGER.info("Download complete. File size: {}", formattedSize);

// Example: Formatting a metric for a debug UI
Metric frameTimeMetric = performanceTracker.getFrameTimeMetric();
String formattedMetric = FormatUtil.simpleFormat(frameTimeMetric);
debugOverlay.setText("Frame Time: " + formattedMetric);
```

### Anti-Patterns (Do NOT do this)
- **Attempted Instantiation:** Never attempt to create an instance of FormatUtil. The class is not designed to be instantiated and lacks a public constructor.
- **Logic Duplication:** Do not write custom formatting logic for time, bytes, or metrics in other parts of the codebase. Always defer to this centralized utility to maintain consistency.
- **Null Inputs:** Most methods are annotated with Nonnull. Passing null arguments will result in a NullPointerException. Always ensure inputs are valid before calling a formatting method.

## Data Pipeline
FormatUtil acts as a terminal transformation stage in a data pipeline, converting raw numerical data into a presentational string format. It does not pass data to other systems; it produces a final output for consumption by users or logs.

> **Flow 1: Performance Monitoring**
> Raw Data (long frameNanos) -> Metric Aggregator -> Metric Object -> **FormatUtil** -> Formatted String -> Debug UI

> **Flow 2: Network Logging**
> Raw Data (int bytesReceived) -> Network Subsystem -> **FormatUtil** -> Formatted String -> Log Output

