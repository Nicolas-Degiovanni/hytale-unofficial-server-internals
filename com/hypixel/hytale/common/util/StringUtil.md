---
description: Architectural reference for StringUtil
---

# StringUtil

**Package:** com.hypixel.hytale.common.util
**Type:** Utility

## Definition
```java
// Signature
public class StringUtil {
```

## Architecture & Concepts

The StringUtil class is a foundational, stateless utility component that provides a centralized suite of high-performance and specialized string manipulation functions. It is designed to be a globally accessible toolkit for common string-related tasks, ensuring consistency and correctness across the entire Hytale codebase.

Its responsibilities can be categorized into several key domains:
*   **Validation:** Provides fast, low-level checks for string content, such as identifying numeric or alphanumeric compositions.
*   **Transformation:** Includes methods for capitalization, quoting, and formatting data types like Duration into human-readable strings.
*   **Parsing:** Offers sophisticated parsing logic for complex structures like command-line arguments, handling quotes, escapes, and named options.
*   **Searching & Matching:** Implements advanced comparison algorithms, including glob pattern matching (*, ?) and fuzzy distance sorting for user-friendly search results.
*   **Visualization:** Contains a powerful text-based graph generator capable of rendering time-series data directly into a StringBuilder, suitable for logs and diagnostic consoles.

This class deliberately avoids maintaining any internal state, ensuring that all its methods behave as pure functions. It leverages the high-performance **fastutil** library for collections in computationally intensive operations like `sortByFuzzyDistance` to minimize memory allocation and CPU overhead.

### Lifecycle & Ownership
- **Creation:** As a static utility class, StringUtil is never instantiated. The class is loaded into the JVM by the system ClassLoader upon its first use.
- **Scope:** Application-wide. Its static methods are available throughout the entire lifecycle of the application.
- **Destruction:** The class is unloaded from the JVM when the application terminates. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** StringUtil is entirely stateless. Its static fields, such as RAW_ARGS_PATTERN, are immutable constants initialized at class-load time. All methods operate exclusively on the arguments provided to them, producing new output without side effects.
- **Thread Safety:** This class is inherently thread-safe. Due to its stateless and functional nature, all methods can be invoked concurrently from any number of threads without risk of race conditions or data corruption. No external synchronization is required by the caller.

## API Surface

The public API provides a range of utilities from simple checks to complex data processing. The following table highlights key methods and their operational complexity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isNumericString(str) | boolean | O(N) | Performs a fast, character-by-character scan of the string. |
| parseEnum(constants, str, matchType) | V | O(N*M) | Searches for an enum constant matching the input string. Complexity depends on the number of enum constants (N) and string length (M). |
| parseArgs(rawString) | String[] | O(N) | Parses a command-line string into an array of arguments. Involves a single pass with state-tracking for quotes and escapes. |
| isGlobMatching(pattern, text) | boolean | O(N*M) | Performs recursive-style pattern matching. Can be computationally expensive for complex patterns and long text. |
| sortByFuzzyDistance(str, collection) | List<T> | O(N log N) | Sorts a collection based on fuzzy string distance. Dominated by the cost of sorting (N) and distance calculation for each item. |
| generateGraph(...) | void | O(W*H) | Generates a text-based graph. Complexity is proportional to the width (W) of the history data and the requested graph dimensions (H). |

## Integration Patterns

### Standard Usage

All methods on this class are static and should be invoked directly. There is no need to obtain an instance from a context or registry.

```java
// Example: Validating a username
String username = "Player_123";
if (StringUtil.isAlphaNumericHyphenUnderscoreString(username)) {
    // Proceed with logic
}

// Example: Parsing console command input
String rawInput = "command \"first arg\" --option=true";
String[] args = StringUtil.parseArgs(rawInput);
```

### Anti-Patterns (Do NOT do this)
- **Attempted Instantiation:** Never attempt to create an instance of StringUtil using `new StringUtil()`. This is not supported and serves no purpose.
- **Logic Re-implementation:** Do not write custom code for functionality already provided by this class, such as argument parsing or glob matching. Using this central utility ensures behavior is consistent system-wide.
- **Misuse in Performance-Critical Loops:** Be cautious when using computationally intensive methods like `isGlobMatching` or `sortByFuzzyDistance` inside tight game loops or on performance-sensitive server threads. Their complexity can introduce significant latency if used with large datasets. Profile their usage carefully.

## Data Pipeline

StringUtil often acts as a transformation or parsing step within a larger data flow. Its role is to convert raw, unstructured string data into a structured format suitable for downstream systems.

> **Flow for Command Parsing:**
> Raw User Input String -> **StringUtil.parseArgs** -> Structured String Array -> Command Dispatcher -> Command Handler

> **Flow for Graph Generation:**
> Time-Series Data Source (e.g., Performance Monitor) -> Functional Callbacks -> **StringUtil.generateGraph** -> Formatted StringBuilder -> Log File / Developer Console

