---
description: Architectural reference for SystemUtil
---

# SystemUtil

**Package:** com.hypixel.hytale.common.util
**Type:** Utility

## Definition
```java
// Signature
public class SystemUtil {
```

## Architecture & Concepts
SystemUtil is a foundational, stateless utility class responsible for abstracting Operating System detection. It serves as a single, immutable source of truth for the runtime environment (Windows, macOS, Linux, or Other).

Its primary architectural role is to decouple higher-level systems from the specifics of JVM system properties. Instead of querying `System.getProperty("os.name")` throughout the codebase, components can rely on the pre-computed and strongly-typed `SystemUtil.TYPE` enum. This centralizes the detection logic, improves readability, and prevents inconsistencies that could arise from string-based comparisons. It is typically used by systems responsible for loading native libraries, handling platform-specific file paths, or applying OS-specific graphical settings.

## Lifecycle & Ownership
As a static utility class, SystemUtil does not follow a traditional object lifecycle. Its state is tied directly to the JVM's class-loading mechanism.

- **Creation:** The class is loaded and initialized by the JVM ClassLoader the first time any code references it, for example, by accessing the `SystemUtil.TYPE` field. During this one-time static initialization, the `getSystemType` method is invoked, and its result is permanently stored in the final static field `TYPE`.
- **Scope:** The state of this class is global and persists for the entire lifetime of the application.
- **Destruction:** The class and its static state are unloaded only when the Java Virtual Machine shuts down. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The class holds a single piece of state: the detected operating system type. This state is **immutable**. It is calculated once during static initialization and cannot be changed for the remainder of the application's execution.
- **Thread Safety:** SystemUtil is inherently **thread-safe**. The JVM guarantees that static class initialization is a synchronous, thread-safe process. Once the `TYPE` field is written, it becomes a read-only constant, which can be safely accessed by any number of threads without synchronization.

## API Surface
The public contract consists of a single static field.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| TYPE | SystemUtil.SystemType | O(1) | A static final enum representing the detected operating system. Access is a direct field read. |

## Integration Patterns

### Standard Usage
SystemUtil is designed for direct static access. It should be used for conditional logic that depends on the host operating system.

```java
// Example: Selecting a platform-specific library path
if (SystemUtil.TYPE == SystemUtil.SystemType.WINDOWS) {
    // Load natives for Windows
} else if (SystemUtil.TYPE == SystemUtil.SystemType.LINUX) {
    // Load natives for Linux
}
```

### Anti-Patterns (Do NOT do this)
- **Repetitive Checks:** Do not query `System.getProperty("os.name")` elsewhere in the code. Always defer to `SystemUtil.TYPE` to maintain a single source of truth.
- **Instantiation:** The class has no public constructor and is not designed to be instantiated. Attempting to create an instance via reflection is an anti-pattern and serves no purpose.
- **String Comparison:** Avoid comparing the enum's name as a string. Use direct enum comparison for type safety and performance.
    - **BAD:** `if (SystemUtil.TYPE.name().equals("WINDOWS"))`
    - **GOOD:** `if (SystemUtil.TYPE == SystemUtil.SystemType.WINDOWS)`

## Data Pipeline
SystemUtil acts as an initial data source, not a processing stage in a pipeline. It performs a one-time transformation of a low-level system property into a high-level, type-safe enum.

> Flow:
> JVM System Property ("os.name") -> **SystemUtil.getSystemType()** -> static final TYPE field

