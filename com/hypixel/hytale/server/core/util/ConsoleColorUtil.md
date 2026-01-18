---
description: Architectural reference for ConsoleColorUtil
---

# ConsoleColorUtil

**Package:** com.hypixel.hytale.server.core.util
**Type:** Utility

## Definition
```java
// Signature
public class ConsoleColorUtil {
```

## Architecture & Concepts
ConsoleColorUtil is a stateless utility class that provides a centralized repository for ANSI escape codes used to color terminal output. Its primary architectural role is to decouple logging and console-related systems from the raw, platform-specific string literals required for text formatting.

By abstracting these values into static constants, it promotes code readability, maintainability, and consistency across all server-side console outputs. It is a foundational, low-level component intended for use by higher-level systems such as the server's primary logging framework, command processors, or diagnostic tools.

## Lifecycle & Ownership
- **Creation:** As a class containing only static final members, ConsoleColorUtil is never instantiated. The class is loaded into the JVM by the ClassLoader upon its first reference, at which point its static fields are initialized.
- **Scope:** The class and its constants persist for the entire lifetime of the server's JVM process.
- **Destruction:** The class is unloaded from memory only when the JVM shuts down. No manual resource management is required.

## Internal State & Concurrency
- **State:** This class is completely stateless. Its members are compile-time constants of type String, which is an immutable type in Java.
- **Thread Safety:** ConsoleColorUtil is inherently thread-safe. Its constants can be safely accessed and read by any number of concurrent threads without locks or synchronization primitives.

## API Surface
The public contract of this class consists exclusively of its static final fields.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BLACK | String | O(1) | ANSI escape code for black text. |
| RED | String | O(1) | ANSI escape code for red text. Typically used for errors. |
| GREEN | String | O(1) | ANSI escape code for green text. Typically used for success messages. |
| YELLOW | String | O(1) | ANSI escape code for yellow text. Typically used for warnings. |
| BLUE | String | O(1) | ANSI escape code for blue text. |
| PURPLE | String | O(1) | ANSI escape code for purple text. |
| CYAN | String | O(1) | ANSI escape code for cyan text. |
| WHITE | String | O(1) | ANSI escape code for white text. Used to reset color to default. |

## Integration Patterns

### Standard Usage
This utility is intended to be used for string concatenation or formatting within systems that write to a compatible terminal.

```java
// Standard pattern for logging a warning message
System.out.println(
    ConsoleColorUtil.YELLOW + 
    "Warning: Configuration value is deprecated." + 
    ConsoleColorUtil.WHITE
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this class. It provides no instance members and doing so is a waste of memory.
  - **Incorrect:** `new ConsoleColorUtil();`
- **Hardcoding ANSI Values:** Avoid using raw ANSI escape codes directly in application code. This defeats the purpose of this utility and leads to code that is difficult to read and maintain.
  - **Incorrect:** `System.out.println("\u001b[0;31m" + "Critical Failure!");`

## Data Pipeline
ConsoleColorUtil does not process data; it is a source of static data used by other components in a pipeline. It injects formatting information into a data stream, typically a string destined for console output.

> Flow:
> Logging System -> **ConsoleColorUtil** (Retrieves constant) -> String Formatter -> Console OutputStream

