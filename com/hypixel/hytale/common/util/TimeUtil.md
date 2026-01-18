---
description: Architectural reference for TimeUtil
---

# TimeUtil

**Package:** com.hypixel.hytale.common.util
**Type:** Utility

## Definition
```java
// Signature
public class TimeUtil {
```

## Architecture & Concepts
TimeUtil is a stateless, static utility class designed to provide robust and safe time-based calculations. Its primary architectural role is to centralize complex temporal logic, specifically the comparison of time differences that may exceed the precision of standard Java time library methods and lead to arithmetic overflows.

This component resides in the common utility layer, making it accessible to both client and server codebases. It serves as a foundational building block for any system that needs to measure or compare very large durations, such as session lengths, long-term cooldowns, or persistent state timers. By providing a single, hardened implementation, it prevents the proliferation of bespoke and potentially buggy time comparison logic across the engine.

The core design centers on a fallback mechanism. It first attempts a high-precision comparison using nanoseconds. If this fails due to an `ArithmeticException` (indicating the duration is too large to be represented in nanoseconds), it gracefully degrades to a lower-precision comparison of seconds, supplemented by a separate nanosecond-of-second check to maintain correctness.

## Lifecycle & Ownership
- **Creation:** As a static utility class, TimeUtil is never instantiated. The Java Virtual Machine loads the class into memory on its first use.
- **Scope:** The class and its static methods are available for the entire lifetime of the application once loaded by the ClassLoader.
- **Destruction:** The class is unloaded from memory when the application's ClassLoader is garbage collected, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** TimeUtil is completely **stateless**. It contains no member variables and all its methods are pure functions, operating exclusively on the arguments provided. The output is solely determined by the input.
- **Thread Safety:** This class is inherently **thread-safe**. Its stateless nature ensures that concurrent calls from multiple threads cannot interfere with each other. No locks or synchronization primitives are required.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| compareDifference(Instant from, Instant to, Duration duration) | int | O(1) | Compares the duration between two Instants against a reference Duration. Returns -1, 0, or 1. This method is overflow-safe and handles extremely large time spans that would cause standard library methods to fail. |

## Integration Patterns

### Standard Usage
This utility should be used whenever comparing the duration between two points in time against a known threshold, especially when the time points could be far apart.

```java
// Example: Checking if a player's ban has expired
Instant banIssuedTime = ...; // From database
Instant currentTime = Instant.now();
Duration banDuration = Duration.ofDays(90);

int comparison = TimeUtil.compareDifference(banIssuedTime, currentTime, banDuration);

if (comparison >= 0) {
    // The time elapsed since the ban is greater than or equal to the ban duration.
    // The ban has expired.
}
```

### Anti-Patterns (Do NOT do this)
- **Using Standard Library for Large Spans:** Avoid using `Duration.between(from, to).compareTo(duration)` directly when `from` and `to` can be separated by many years. The intermediate `Duration` object created by `between` can overflow, throwing an exception that crashes the calling thread. TimeUtil is specifically designed to prevent this.
- **Re-implementing Fallback Logic:** Do not write custom logic to catch `ArithmeticException` from time calculations. This utility provides a tested and centralized solution.

## Data Pipeline
TimeUtil acts as a pure transformation function, not as a stage in a larger data processing pipeline. Its data flow is simple and immediate.

> Flow:
> (Input) Instant, Instant, Duration -> **TimeUtil.compareDifference** -> (Output) int

